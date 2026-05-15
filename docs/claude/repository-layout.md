# Repository Layout

> Six repositories under the INENI-PT-GROUP-B organization. This document
> describes each repo's purpose, key contents, and how they fit together.

## Overview

The platform spans six repositories. Five are public; `app-frontend` is
private (per the original assignment requirement, retained for git-history
visibility after the pivot to Gitea). All repositories follow the
conventions in [`CONTRIBUTING.md`](../../CONTRIBUTING.md).

| Repo               | Visibility | Purpose                              |
| ------------------ | ---------- | ------------------------------------ |
| `platform`         | public     | Docs hub, conventions, this context  |
| `platform-iac`     | public     | Terraform for GCP infrastructure     |
| `platform-gitops`  | public     | Argo CD + Crossplane + Helm values   |
| `.github`          | public     | Org-wide defaults                    |
| `app-backend`      | public     | Archived after pivot to Gitea        |
| `app-frontend`     | private    | Archived after pivot to Gitea        |

## platform

The entry point of the project. New readers (lecturer, future team members)
start here.

Contents:
- `README.md` — project overview, links to all repos and key docs
- `CONTRIBUTING.md` — canonical conventions for the entire org
- `AI_USAGE.md` — log of generative AI usage across the project
- `scripts/bootstrap.sh` — Day-1 orchestration: `terraform apply` →
  kubeconfig → Argo CD root sync. See
  [`architecture-decisions.md`](./docs/claude/architecture-decisions.md#bootstrap-orchestration)
- `docs/cost-planning/` — capacity and cost estimates with dated snapshots
- `docs/claude/` — context bundle for AI-assisted work (this folder)
- `docs/architecture/` — diagrams, design notes (added as needed)

This repo contains no application code, only documentation, the bootstrap
script, and shared agreements.

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

Originally planned as the application's REST API. After the pivot to Gitea
(see [`architecture-decisions.md`](./architecture-decisions.md#the-application)),
this repo is archived. No code we own lives here. Retained for git-history
visibility — the earlier fork of the lecturer's reference application is the
last commit on `main`.

## app-frontend

Originally planned as the application's single-page frontend. After the
pivot to Gitea, this repo is archived. No code we own lives here. Retained
for git-history visibility. The repo remains private (per original
assignment requirement); lecturer (`@muhlba91`) retains admin access.

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
                   Argo CD Application, which deploys Gitea
                   via the wrapper chart (charts/tenant-app/)
                   referencing the upstream Helm chart at
                   oci://docker.gitea.com/charts/gitea
```

**Adding a new tenant** is a single PR to `platform-gitops/tenants/`. No
infrastructure change, no IaC re-run.

**Updating a platform component** (e.g. bumping cert-manager) is a single PR
to `platform-gitops/applications/` or `platform-gitops/values/`. Argo CD
reconciles after merge.

**Updating the cluster topology** (e.g. adding a node pool) is a PR to
`platform-iac`, applied after review.

**Application version changes** (Gitea release upgrades) are made via a
single PR in `platform-gitops` bumping the chart-dependency version in
`charts/tenant-app/Chart.yaml`. After `helm dependency update` and merge,
Argo CD reconciles the new Gitea version across all tenant namespaces —
satisfying the assignment requirement "updates rolled out to all tenants
with a single change".

## Cross-repo references in commits and PRs

Use the `<owner>/<repo>#<number>` syntax. Example:

```
fix(crossplane): handle DB connection retry on tenant create

Refs INENI-PT-GROUP-B/platform-gitops#42
Closes INENI-PT-GROUP-B/platform#7
```
