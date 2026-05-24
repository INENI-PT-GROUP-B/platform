# Repository Layout

> Six repositories under the INENI-PT-GROUP-B organization. This document
> describes each repo's purpose, key contents, and how they fit together.

## Overview

The platform spans six repositories. Five are public; only `app-frontend` is
private (per assignment requirement). All repositories follow the conventions
in [`CONTRIBUTING.md`](../../CONTRIBUTING.md).

| Repo               | Visibility | Purpose                              |
| ------------------ | ---------- | ------------------------------------ |
| `platform`         | public     | Docs hub, conventions, this context  |
| `platform-iac`     | public     | Terraform for GCP infrastructure     |
| `platform-gitops`  | public     | Argo CD + Crossplane + Helm values   |
| `.github`          | public     | Org-wide defaults                    |
| `app-backend`      | public     | Application backend + Helm chart     |
| `app-frontend`     | private    | Application frontend (SPA)           |

## platform

The entry point of the project. New readers (lecturer, future team members)
start here.

Contents:
- `README.md` — project overview, links to all repos and key docs
- `CONTRIBUTING.md` — canonical conventions for the entire org
- `AI_USAGE.md` — log of generative AI usage across the project
- `docs/cost-planning/` — capacity and cost estimates with dated snapshots
- `docs/claude/` — context bundle for AI-assisted work (this folder)
- `docs/architecture/` — diagrams, design notes (added as needed)

This repo contains no code, only documentation and shared agreements.

## platform-iac

Terraform code for the Day 1 bootstrap. Applied **locally** via
`bootstrap/bootstrap.sh` — a single idempotent script that creates the
state bucket in its preflight phase, runs `terraform apply`, then
installs Argo CD with the root App-of-Apps pointing at
`platform-gitops`. After that one script run, Argo CD reconciles
everything else from `platform-gitops`.

Current state (early bootstrap phase):
- `DNS_SETUP.md` — notes for the manually-created Cloud DNS zone (used
  to unblock NS-delegation). The zone will be brought under Terraform
  management once the `dns/` IaC code lands; the exact mechanism is
  still to be determined.
- `.github/workflows/` — CI scaffolding (commitlint, PR title
  validation)
- `.commitlintrc.json`, `.gitignore` — repo hygiene

Planned structure (added as IaC code is written):
- `bootstrap/bootstrap.sh` — the idempotent entry point that
  orchestrates the phases (preflight, API enablement, state bucket,
  `terraform apply`, kubeconfig, Argo CD bootstrap)
- Terraform root modules per concern, e.g. `network/`, `cluster/`,
  `iam/`, `dns/`
- `backend.tf` — remote state in the `<project>-tfstate` GCS bucket
  created by the bootstrap script
- `modules/` — reusable in-repo modules

Authentication: Terraform runs as the executing team member via
`gcloud auth login`. No CI-based Terraform pipeline, no GitHub Actions
OIDC ↔ GCP trust binding, no long-lived JSON keys.

## platform-gitops

Source of truth for everything Argo CD reconciles into the cluster. Argo CD
itself bootstraps to this repo from `platform-iac`. From that point on, all
changes to the platform happen through PRs in this repo.

Directory layout (already established):

| Path                         | Purpose                                                         |
| ---------------------------- | --------------------------------------------------------------- |
| `applications/`              | Argo CD `Application` manifests for platform components         |
| `applicationsets/`           | `ApplicationSet` manifests; currently reserved — tenant apps are deployed by Crossplane via `provider-helm`, not via ApplicationSets |
| `crossplane/providers/`      | Crossplane provider configurations                              |
| `crossplane/xrds/`           | Composite Resource Definitions (XRDs)                           |
| `crossplane/compositions/`   | Compositions implementing the XRDs                              |
| `tenants/`                   | Tenant **claims** — one file or folder per tenant (including a dedicated `staging.yaml`) |
| `values/`                    | Helm values for platform components, plus `app-version.yaml` (central image tags + chart version for the tenant app) |
| `.github/workflows/`         | CI for `yamllint`, `helm lint`, `kubeconform` on PRs            |

`values/app-version.yaml` schema:

```yaml
chart:
  version: "0.1.0"
images:
  backend: "v0.1.0"
  frontend: "v0.1.0"
```

A tenant claim can override these on a per-tenant basis via
`spec.imageTagOverride.backend` and `spec.imageTagOverride.frontend`.
The dedicated `staging` tenant uses this for new-version testing.

Onboarding flow:

1. PR adds a Tenant claim under `tenants/`.
2. Argo CD reconciles the claim into the cluster as a custom resource.
3. Crossplane picks up the claim and, via its Composition, provisions
   the per-tenant infrastructure (namespace, DB, network policies, quotas,
   GHCR pull secret, Traefik BasicAuth middleware and its backing Secret)
   and the tenant app as a Helm `Release` rendered by `provider-helm`
   (chart pulled from GHCR as an OCI artefact).
4. Tenant is live.

## .github

Org-wide defaults. Files in this repo propagate to all other repos in the org
(profile page, issue templates, PR template).

Contents:
- `.github/PULL_REQUEST_TEMPLATE.md` — PR template referenced by `CONTRIBUTING.md`
- `.github/workflows/commitlint-reusable.yml` — reusable commitlint workflow
- `.github/workflows/pr-title-reusable.yml` — reusable PR title validation
- `.commitlintrc.json` — Conventional Commits config

Each repo in the org imports the reusable workflows via:

```yaml
uses: INENI-PT-GROUP-B/.github/.github/workflows/commitlint-reusable.yml@main
```

## app-backend

The application's REST API plus the Helm chart that deploys the full tenant
app (backend + frontend + ingress). Public repository, public GHCR image
(`ghcr.io/ineni-pt-group-b/app-backend`), public GHCR Helm chart
(`ghcr.io/ineni-pt-group-b/app-chart`).

Contents:
- Application source code (Node.js, TypeScript, Fastify, Drizzle ORM)
- `Dockerfile` — multi-stage Alpine build
- `chart/` — Helm chart deploying backend + frontend + ingress for one
  tenant. CloudNativePG and the Traefik BasicAuth middleware are not part
  of this chart; both are produced by the Crossplane Composition directly.
  Chart layout:
  ```
  chart/
    Chart.yaml
    values.yaml
    templates/
      backend-deployment.yaml
      backend-service.yaml
      frontend-deployment.yaml
      frontend-service.yaml
      frontend-config-cm.yaml
      ingress.yaml
      _helpers.tpl
  ```
- `.github/workflows/ci.yaml` — lint, test on PRs
- `.github/workflows/release.yaml` — on release tag, builds and pushes the
  backend image, then packages and `helm push`-es the chart as an OCI
  artefact to `ghcr.io/ineni-pt-group-b/app-chart`
- `README.md` — local dev setup, env-var contract, API description

App contract: see
[`architecture-decisions.md`](./architecture-decisions.md#the-application).

## app-frontend

The application's single-page frontend. **Private** repository, private GHCR
image (`ghcr.io/ineni-pt-group-b/app-frontend`).

Contents:
- Application source code (Vite + React 18 + TypeScript, React Router,
  TanStack Query, Tailwind CSS)
- `Dockerfile` — multi-stage build with nginx runtime on port 80 serving the
  static bundle. `/config.js` is delivered by the Helm chart as a ConfigMap
  (`window.APP_CONFIG`, `apiBaseUrl` always `/api`); the image serves the file
  statically and must not also write it
- `.github/workflows/ci.yaml` — lint, build on PRs
- `.github/workflows/release.yaml` — on release tag, builds and pushes the
  frontend image only (no chart in this repo)
- `README.md` — local dev setup, env-var contract

The image-pull secret for GHCR is synced per tenant namespace via ESO.
Lecturer (`@muhlba91`) has admin access to this private repo.

## How the repos fit together

The deployment chain runs in one direction:

```
platform-iac     →  GKE cluster + Argo CD installed
                          ↓
                   Argo CD points at platform-gitops
                          ↓
platform-gitops  →  Platform components installed
                    (ESO, ExternalDNS, cert-manager, Traefik, Crossplane,
                    CloudNativePG, Prometheus, Grafana,
                    plus Argo CD self-managing from here on)
                          ↓
platform-gitops/tenants/  →  Tenant claim added
                          ↓
                   Crossplane provisions per-tenant resources, including
                   the Traefik BasicAuth middleware, and renders the app
                   Helm chart (pulled from GHCR as an OCI artefact) into
                   the tenant namespace via provider-helm
```

**Adding a new tenant** is a single PR to `platform-gitops/tenants/`. No
infrastructure change, no IaC re-run.

**Updating a platform component** (e.g. bumping cert-manager) is a single PR
to `platform-gitops/applications/` or `platform-gitops/values/`. Argo CD
reconciles after merge.

**Updating the cluster topology** (e.g. adding a node pool) is a PR to
`platform-iac`, applied after review.

**Application code changes** in `app-backend` produce a new backend image
and a new chart version (both tagged identically). **Application code changes**
in `app-frontend` produce a new frontend image only. The rollout to all
tenants happens via a single change in `platform-gitops/values/app-version.yaml`
updating the relevant tags and chart version — satisfying the assignment
requirement "updates rolled out to all tenants with a single change".

## Cross-repo references in commits and PRs

Use the `<owner>/<repo>#<number>` syntax. Example:

```
fix(crossplane): handle DB connection retry on tenant create

Refs INENI-PT-GROUP-B/platform-gitops#42
Closes INENI-PT-GROUP-B/platform#7
```
