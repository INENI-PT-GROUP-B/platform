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
| `app-backend`      | public     | Application backend (REST API)       |
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

Terraform code for the Day 1 bootstrap. Once `terraform apply` runs against
this repo, the GKE cluster exists, Argo CD is installed, and Argo CD points
at `platform-gitops`.

Current state (early bootstrap phase):
- `DNS_SETUP.md` — manual setup notes for the Cloud DNS zone (the zone was
  created out-of-band to unblock NS-delegation; will be `terraform import`-ed
  once IaC code lands)
- `.github/workflows/` — CI scaffolding (commitlint, PR title validation)
- `.commitlintrc.json`, `.gitignore` — repo hygiene

Planned structure (added as IaC code is written):
- Terraform root modules per concern, e.g. `cluster/`, `dns/`, `iam/`, `argocd/`
- `backend.tf` — remote state in a GCS bucket (versioned, with state locking)
- `modules/` — reusable in-repo modules

Authentication: GitHub Actions OIDC → GCP Workload Identity Federation. No
long-lived JSON keys.

## platform-gitops

Source of truth for everything Argo CD reconciles into the cluster. Argo CD
itself bootstraps to this repo from `platform-iac`. From that point on, all
changes to the platform happen through PRs in this repo.

Directory layout (already established):

| Path                         | Purpose                                                         |
| ---------------------------- | --------------------------------------------------------------- |
| `applications/`              | Argo CD `Application` manifests for platform components         |
| `applicationsets/`           | `ApplicationSet` manifests that template tenant Applications    |
| `crossplane/providers/`      | Crossplane provider configurations                              |
| `crossplane/xrds/`           | Composite Resource Definitions (XRDs)                           |
| `crossplane/compositions/`   | Compositions implementing the XRDs                              |
| `tenants/`                   | Tenant **claims** — one file or folder per tenant               |
| `values/`                    | Helm values for platform components                             |
| `.github/workflows/`         | CI for `yamllint`, `helm lint`, `kubeconform` on PRs            |

Onboarding flow:

1. PR adds a Tenant claim under `tenants/`.
2. Argo CD reconciles, Crossplane picks up the claim and provisions the
   per-tenant infrastructure via the matching Composition.
3. The relevant ApplicationSet emits an Argo CD Application for the tenant's
   app deployment.
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

The application's REST API. Public repository, public GHCR image
(`ghcr.io/ineni-pt-group-b/app-backend`).

Contents:
- Application source code
- `Dockerfile` — multi-stage build, Alpine-based
- `.github/workflows/` — CI for lint, test, image build, image push to GHCR
  on tag or release
- `README.md` — local dev setup, env-var contract, API description

App contract: see
[`architecture-decisions.md`](./architecture-decisions.md#the-application).

> **Note:** at the time of writing, this repo is still a fork of the lecturer's
> reference application. It will be replaced with our own minimal application
> (delete fork, create fresh repo) before we start the application management
> pillar.

## app-frontend

The application's single-page frontend. **Private** repository, private GHCR
image (`ghcr.io/ineni-pt-group-b/app-frontend`).

Contents: analogous to `app-backend` (source, Dockerfile, CI workflows, README).

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
                    (ESO, ExternalDNS, cert-manager, Crossplane,
                    CloudNativePG, Prometheus, Grafana,
                    plus Argo CD self-managing from here on)
                          ↓
platform-gitops/tenants/  →  Tenant claim added
                          ↓
                   Crossplane provisions per-tenant resources
                          ↓
                   ApplicationSet generates the tenant's
                   Argo CD Application, pulling app-backend
                   and app-frontend images from GHCR
```

**Adding a new tenant** is a single PR to `platform-gitops/tenants/`. No
infrastructure change, no IaC re-run.

**Updating a platform component** (e.g. bumping cert-manager) is a single PR
to `platform-gitops/applications/` or `platform-gitops/values/`. Argo CD
reconciles after merge.

**Updating the cluster topology** (e.g. adding a node pool) is a PR to
`platform-iac`, applied after review.

**Application code changes** (`app-backend`, `app-frontend`) trigger an image
rebuild and produce a new image tag. The rollout to all tenants happens via a
single change in `platform-gitops` updating the image tag — satisfying the
assignment requirement "updates rolled out to all tenants with a single
change".

## Cross-repo references in commits and PRs

Use the `<owner>/<repo>#<number>` syntax. Example:

```
fix(crossplane): handle DB connection retry on tenant create

Refs INENI-PT-GROUP-B/platform-gitops#42
Closes INENI-PT-GROUP-B/platform#7
```
