# Project Overview

> Context document for AI-assisted work. Load this file into every Claude session
> alongside the other files in this folder.

## What we are building

A Kubernetes-based platform that provisions isolated, multi-tenant SaaS application
instances on demand. Each tenant receives their own namespace with a dedicated
application stack (frontend, backend, database) provisioned via GitOps and
Crossplane.

This is the assignment for the **Infrastructure Engineering** course at
Hochschule Burgenland, summer term 2026.

## Team

- Organization: [INENI-PT-GROUP-B](https://github.com/INENI-PT-GROUP-B)
- Members (4): Marco (`mr`), Alex (`am`), Ronny (`rl`), Patrick (`pp`)
- Lecturer: Daniel Mühlbachler-Pietrzykowski (`@muhlba91`) — has admin access
  to all repositories

## Deadline

**2026-06-26 14:00 CEST** — submission deadline. No changes permitted afterwards.
The project concludes with a 20-minute presentation and demo.

## Grading pillars

The project is graded across four pillars. All must be passed individually.

| Pillar                                       | Weight |
| -------------------------------------------- | ------ |
| Documentation & Software Management Hygiene  | 15%    |
| Infrastructure Bootstrap                     | 35%    |
| Application Management                       | 35%    |
| Presentation                                 | 15%    |

The presentation grade applies equally to all team members. The other three
pillars require evidence of equal contribution per member, traced via GitHub
usernames (commits, PRs, issues). Inequalities will lead to deductions.

## Hard constraints

These rules are non-negotiable and must be reflected in every decision:

- **All repositories are public** except `app-frontend` (which must be private
  per assignment requirement). Treat everything as world-readable.
- **No plaintext or hardcoded secrets** anywhere — not in code, not in commits,
  not in CI logs, not in Helm values. Use External Secrets Operator (ESO)
  with Google Secret Manager.
- **No direct commits to `main`.** All changes go through pull requests.
  See [`CONTRIBUTING.md`](../../CONTRIBUTING.md) for the full workflow.
- **No manual click after `bootstrap.sh` kickoff.** Once a team member runs
  `bootstrap/bootstrap.sh` locally, the script provisions the entire platform
  end-to-end (state bucket, GKE cluster, IAM, Argo CD bootstrap) and then
  Argo CD reconciles everything else from `platform-gitops`. The single
  manual invocation of the script is the only documented exception per the
  assignment.
- **No long-lived GCP service account JSON keys** anywhere — not on team
  members' machines, not in CI, not in cluster Secrets. Terraform runs
  locally as the executing team member, authenticated via `gcloud auth
  login`. In-cluster workloads use GKE Workload Identity. CI workloads
  that need GCP access would use GitHub Actions OIDC → Workload Identity
  Federation, but **no such CI workload currently exists** — GitHub
  Actions only runs PR validation and GHCR image/chart pushes (the latter
  authenticated via the built-in `GITHUB_TOKEN`).
- **Conventional Commits + squash merge.** No merge commits. Linear history.
- **English only.** All code, commits, PRs, issues, documentation, and code
  comments are written in English. No mixed-language artefacts.
- **No AI attribution in git history.** Claude must not add Co-Authored-By
  trailers, footers, or attribution comments to commits, PRs, or code.
  AI usage is tracked centrally in `AI_USAGE.md`.
- **AI usage must be disclosed.** See [`working-with-claude.md`](./working-with-claude.md)
  and the project-wide `AI_USAGE.md`.

## Day 1 vs Day 2 separation

The assignment distinguishes two phases. We follow this split to avoid
overengineering.

**Day 1 — Foundation (Bootstrap).** Provisioning the GKE cluster, VPC, Workload
Identity, and the GitOps tool. Result: the platform exists and is ready to host
tenants. Implemented via Terraform in `platform-iac`, executed locally through
`bootstrap/bootstrap.sh`.

**Day 2 — Service Catalog (Application).** On-the-fly provisioning of
tenant-specific application instances. A developer triggers a deployment via
GitOps, Crossplane translates that into the underlying infrastructure (database,
namespace, network policies, application Helm release). Implemented in
`platform-gitops` with Crossplane Compositions.

## Where to find what

- [`architecture-decisions.md`](./architecture-decisions.md) — chosen tools and why
- [`repository-layout.md`](./repository-layout.md) — six repos and their purposes
- [`conventions.md`](./conventions.md) — commit/PR/branch cheat-sheet (canonical: [`CONTRIBUTING.md`](../../CONTRIBUTING.md))
- [`tooling-reference.md`](./tooling-reference.md) — distilled lecture content for tools we use
- [`working-with-claude.md`](./working-with-claude.md) — how the team uses Claude

## Out of scope

To stay focused, we explicitly do not pursue:

- Continuous Delivery via Argo Rollouts or Kargo (bonus task, skipped)
- Frontend builds to GCS bucket via Crossplane (bonus task, skipped)
- Kyverno policies and reports (bonus task, skipped)
- Hard multi-tenancy or virtual clusters — soft multi-tenancy is sufficient

The only bonus task we pursue is **basic monitoring** (Prometheus + Grafana
self-hosted in-cluster).
