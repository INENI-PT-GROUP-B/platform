# AI Usage Log

Significant uses of generative AI on this project, in reverse chronological
order. Required by the course assignment and by our
[`CONTRIBUTING.md`](./CONTRIBUTING.md#ai-usage-disclosure).

## When to add an entry

The simple test: **would you mention this if the lecturer asked at the demo
"where did you use AI?"** If yes, log it. If no, skip it.

Concretely: log substantial generated components (modules, charts,
compositions, application files), AI-influenced design decisions, and
notable AI-assisted debugging. Skip autocomplete, syntax lookups, comment
polishing, and explanations that never reach the repo.

When in doubt, log it.

## Entry format

```
## YYYY-MM-DD — <short title>

- **Tool:** <name and version where relevant>
- **Scope:** <repo / component / files>
- **What:** <what was generated or assisted>
- **Verification:** <how it was reviewed or tested>
- **Outcome:** <merged / partial / discarded>
```

Each contributor adds their own entries, ideally in the same PR as the
AI-assisted change.

---

## 2026-05-29 — Argo CD bootstrap (S2-01)

- **Tool:** Claude Code (local, macOS, Opus 4.7)
- **Scope:** `platform-iac` — new `bootstrap/argocd-values.yaml` and
  `bootstrap/argocd-bootstrap.yaml`; `bootstrap/bootstrap.sh` (new Phase 5);
  `README.md`
- **What:** drafted the Argo CD bootstrap that fills the Phase-5 placeholder
  left by S1-10 — a minimal Helm values file (server insecure behind Traefik,
  `argocd.fhuebung.lol` Ingress with the wildcard TLS secret
  `wildcard-fhuebung-lol`), a single self-adopting root App-of-Apps pointing at
  `platform-gitops/applications/`, and Phase 5 in `bootstrap.sh` (pinned chart
  `argo/argo-cd` 9.5.16, `helm upgrade --install` then a `kubectl apply` of the
  root). Only values diverging from the chart defaults were set; the chart
  version was verified and pinned.
- **Verification:** the chart was rendered with `helm template` against the
  values file to confirm the Ingress class, host, wildcard TLS secret and
  `server.insecure`; `bash -n` and a YAML parse passed. Review feedback from
  `@marco93r` was incorporated — a redundant `kubectl rollout status` was
  removed (`helm --wait` already covers it) and the root apply was switched to
  server-side so Argo CD takes over field ownership cleanly on first sync.
  `@mlexinho27` flagged the wildcard-cert producer and its namespace as a
  non-blocking cross-component follow-up, to be handled on the cert-manager
  side. End-to-end run against the cluster is deferred to Day-1 validation.
- **Outcome:** merged — `platform-iac`#40 (closes #38).

## 2026-05-25 — Crossplane provider configs, ExternalDNS, pg-backups IAM hardening

- **Tool:** Claude Code (local, WSL, Opus 4.7)
- **Scope:** `platform-gitops` — `crossplane/providers/` (three ProviderConfigs
  + three DeploymentRuntimeConfigs) and
  `applications/crossplane-providerconfigs.yaml` (#12, S2-10); ExternalDNS
  `applications/externaldns.yaml` + `values/externaldns.yaml` (#14, S2-04).
  `platform-iac` — `terraform/backup/` least-privilege custom role
  (#37, S1-08a follow-up)
- **What:** generated the Crossplane provider configuration (provider-gcp via
  Workload Identity / `InjectedIdentity`, provider-helm and provider-kubernetes
  in-cluster, plus DeploymentRuntimeConfigs pinning each controller
  ServiceAccount) and a multi-source Argo CD Application for ExternalDNS (Cloud
  DNS, WI, `txtOwnerId=gke-prod`, `policy=sync`). Also refactored the pg-backups
  bucket IAM from `roles/storage.admin` to a custom role with only
  `storage.buckets.get/setIamPolicy` after review feedback. Resource shapes and
  the external-dns chart values were verified against current upstream docs
  before writing; chart and tool versions pinned.
- **Verification:** YAML validated locally (parse, yamllint limits,
  apiVersions/kinds against upstream docs); `terraform fmt` + `validate` for the
  IaC change. Review feedback — least-privilege scoping, premature issue-closing
  on runtime-gated acceptance criteria, and missing DeploymentRuntimeConfigs for
  helm/kubernetes — was incorporated. Runtime verification (providers Healthy,
  ExternalDNS writing records) is gated on the cluster + root App-of-Apps and
  deferred.
- **Outcome:** merged — `platform-gitops`#12 (closes #11), `platform-iac`#37
  (closes #36). Open — `platform-gitops`#14 (closes #10), pending review.

## 2026-05-25 — app-frontend release workflow (S1-12)

- **Tool:** Claude Code (local, macOS, Opus 4.7)
- **Scope:** `app-frontend` — new `.github/workflows/release.yml`;
  `platform` — `docs/claude/repository-layout.md` § app-frontend (two
  small follow-up doc syncs to keep the canonical layout description in
  step with the workflow)
- **What:** drafted the release workflow for `app-frontend` — GHCR
  image build and push on `push` to `main` and on git tags `v*`, using
  `docker/metadata-action` for tag computation (`sha-<short>` on every
  build, semver `v*` on tag push) and `docker/build-push-action` with
  the GHA cache. Auth via the built-in `GITHUB_TOKEN`. OCI source label
  links the package to the private repo so visibility is inherited.
- **Verification:** a review pass against the project conventions caught
  a real bug — the initial `pattern={{version}}` would have stripped the
  `v` prefix and produced an image tag (`0.1.0`) that did not match the
  `v0.1.0` form pinned in `values/app-version.yaml` per
  `architecture-decisions.md`; replaced with `pattern={{raw}}` before
  opening the PR. After review feedback from `@ronaldley`, the `latest`
  tag was dropped as well — nothing in the platform consumes it (tenants
  pin `v*`, `staging` uses `imageTagOverride`) and a mutable tag cuts
  against the pinning line followed elsewhere. CI green on every push.
- **Outcome:** merged — `app-frontend`#13 (workflow); `platform`#64
  (initial doc sync) and `platform`#67 (doc follow-up after the `latest`
  drop).

## 2026-05-24 — app-frontend SPA (skeleton + property CRUD screens)

- **Tool:** Claude (claude.ai web, Opus 4.7) for planning and brainstorming;
  Claude Code (local, WSL, Opus 4.7) in plan mode at high effort for the
  implementation, a consistency review, and branch / commit / PR drafting
- **Scope:** `app-frontend` — the SPA built across two PRs: skeleton (#7 — Vite
  + React 18 + TypeScript, Tailwind, React Router, TanStack Query, the
  `/config.js` runtime-config loader, nginx Dockerfile, CI) and the property
  CRUD screens (#9 — list, shared create/edit form, confirm dialog, API client +
  TanStack Query hooks against `/api/properties`)
- **What:** the app-frontend was developed collaboratively — Claude drafted the
  implementation against the agreed contract while the team member steered the
  decisions (npm over pnpm, ConfigMap-delivered config.js, no tests for now),
  reviewed, and ran it. Tool/runtime versions were pinned to exact,
  live-verified releases.
- **Verification:** lint + build green; the app was tested locally end-to-end
  against the backend (Postgres + Vite dev proxy). The first delete failed with
  a 400; manual troubleshooting traced it to the client sending a JSON
  content-type on the body-less DELETE request, which was corrected
  (content-type only when a body is present). After the fix the full create /
  edit / delete flow worked. An AI consistency review additionally added a NaN
  guard on the form.
- **Outcome:** merged — app-frontend#7 (skeleton) and #9 (property CRUD screens).

## 2026-05-24 — app-backend skeleton, properties API, and Helm chart (#32)

- **Tool:** Claude (claude.ai web, Opus 4.7) for planning and brainstorming;
  Claude Code (local, WSL, Opus 4.7) in plan mode at high effort for the
  implementation, the consistency review, and branch / commit / PR drafting
- **Scope:** `app-backend` — application skeleton (`src/server.ts`,
  `package.json`, `tsconfig.json`, `eslint.config.js`, `drizzle.config.ts`,
  `Dockerfile`, `docker-compose.yml`, `.github/workflows/ci.yaml`), the
  properties domain (`src/db/schema.ts`, `src/db/index.ts`, `src/routes/`,
  generated migration, Vitest tests), and the Helm chart (`chart/`)
- **What:** generated the Fastify + TypeScript + Drizzle backend in three
  increments — skeleton; the `properties` schema + migration-on-startup +
  `/healthz` + CRUD with validation + tests; and the tenant app Helm chart
  (backend/frontend/ingress, Traefik path routing, conditional BasicAuth
  middleware annotation). Tool and runtime versions were pinned to exact,
  live-verified releases.
- **Verification:** lint + build + a manual end-to-end run against a local
  Postgres (migrations on startup, `/healthz` 200, CRUD round-trip, 400/404
  cases); `helm lint` + `helm template` for the chart. A separate AI
  consistency-review pass against the deployment contract confirmed the
  BasicAuth handling and surfaced fixes (startupProbe, frontend-only image
  pull secret, `required` value guards), which were applied.
- **Outcome:** open as `app-backend`#6 (skeleton), #7 (properties/CRUD/health),
  #8 (Helm chart) — stacked, refs `platform`#32; pending review.

## 2026-05-24 — Terraform IAM module (S1-08)

- **Tool:** Claude Code (local, macOS, Opus 4.7)
- **Scope:** `platform-iac` — new `terraform/iam/` module (`main.tf`,
  `variables.tf`, `outputs.tf`, `versions.tf`), `terraform/main.tf`,
  `terraform/outputs.tf`, `README.md`
- **What:** drafted the Terraform IAM module from the task brief — four
  Google Service Accounts (ExternalDNS, cert-manager, ESO, Crossplane
  provider-gcp), each bound to its in-cluster KSA via Workload Identity,
  plus the two project-level Secret Manager roles ESO and Crossplane need.
  Module is wired into the root with the GSA emails re-exported for
  S1-09 / S2-05 / S2-10. Layout mirrors `terraform/network/` and
  `terraform/cluster/`.
- **Verification:** review pass against the project conventions caught an
  over-permissioned ESO role (`secretmanager.viewer`) and two unused
  outputs; both removed in a follow-up commit. The task brief listed
  `roles/secretmanager.secretVersionAccessor`, which is not a real GCP
  role — replaced with the canonical `roles/secretmanager.secretAccessor`
  and noted in the PR. All `terraform-lint.yml` checks green.
- **Outcome:** open as `platform-iac`#24, closes #23, pending review.

## 2026-05-24 — Reusable lint workflow and linter version pinning

- **Tool:** Claude Code (local, Windows, Opus 4.7)
- **Scope:** `.github` — `.github/workflows/lint-reusable.yml`, `README.md`;
  `app-backend` / `app-frontend` — `.github/workflows/lint.yml`
- **What:** addressed the review on `app-backend`#3 / `app-frontend`#3 by
  pinning the `yamllint` and `markdownlint-cli2` versions, then extracted the
  duplicated lint setup into a reusable `lint-reusable.yml`
  (`on: workflow_call`) in `.github`, mirroring the existing commitlint /
  pr-title reusables, and documented in the repo README that each consumer
  supplies its own `.yamllint.yml` / `.markdownlint.jsonc` via the caller
  checkout.
- **Verification:** confirmed both pinned versions exist and are current on
  PyPI / npm before pinning; checked the org `main-protection` ruleset
  enforces no named status checks, so the reusable-call check rename is safe;
  cross-checked the caller-checkout config mechanism against the working
  commitlint reusable; CI green on every PR.
- **Outcome:** pinning merged (`app-backend`#3, `app-frontend`#3); reusable
  workflow open as `.github`#9 (refs `.github`#8), consumer migrations to
  follow once it merges.

## 2026-05-21 — Task-list redesign and repo creation via Claude Code

- **Tool:** Claude Code (local, WSL, Opus 4.7, high effort)
- **Scope:** `platform` — `tasks/task-list.md`; GitHub org —
  `app-backend` / `app-frontend` repository creation and settings
- **What:** redesigned the project task list and clarified the sprint
  sequencing and dependency plan, incorporating the team's feedback;
  and tested creating the two application repositories directly through
  Claude Code (repository creation, squash-only merge settings, and the
  `main-protection` branch-protection ruleset).
- **Verification:** repository creation through Claude Code worked well;
  confirmed — as already seen on the public repos — that branch
  protection / rulesets cannot be enabled on a private repository on the
  GitHub Free plan, so `app-frontend` stays convention-only.
- **Outcome:** task list committed via PR #41 (closes #40); app repos
  created with an active ruleset on `app-backend` and convention-only
  protection on `app-frontend`.

## 2026-05-21 — Context-file updates after team architecture decisions (PR #34, issue #33)

- **Tool:** Claude (claude.ai web, Opus 4.7) for the document and diagram
  updates; Claude Code (local, WSL) for content review, branch / commit /
  PR drafting, and `gh` operations
- **Scope:** `platform` — the five `docs/claude/` and `docs/architecture/`
  documents plus `logical_architecture.drawio` (+ PNG export)
- **What:** updated the markdown context files and the logical diagram to
  reflect the minor architectural decisions taken within the team:
  ingress-level Traefik BasicAuth, reduced backend env contract
  (`DATABASE_URL`, `PORT`), Helm chart in `app-backend/chart/`,
  `values/app-version.yaml` schema, and local `bootstrap.sh` (no CI
  Terraform, no GitHub Actions ↔ GCP trust binding).
- **Verification:** Claude Code checked the changes against #33's acceptance
  criteria, confirmed the BasicAuth chain, app-version schema and
  `bootstrap.sh` read consistently across files, flagged and neutralised an
  out-of-scope DNS claim, and re-confirmed the PNG matches the `.drawio`
  source.
- **Outcome:** PR #34 (closes #33), pending review.

## 2026-05-20 — Application brainstorming and decision-making (issues #21 → #32)

- **Tool:** Claude (claude.ai web, Opus 4.7)
- **Scope:** `platform` — application design decisions; led to closing #21
  and opening implementation issue #32
- **What:** brainstormed the application development direction and worked
  through the key decisions: dropping application-level authentication in
  favour of ingress-level Traefik BasicAuth, hard-resetting the
  `app-backend` / `app-frontend` repositories, locating the Helm chart in
  `app-backend/chart/`, and running Terraform locally via `bootstrap.sh`
  instead of from CI.
- **Verification:** decisions cross-checked against the existing deployment
  contract and the platform architecture before being recorded; captured as
  a closure comment on #21 and the scope of #32.
- **Outcome:** decisions recorded — #21 closed, #32 opened for implementation.

## 2026-05-19 — Architecture refinements from lecturer feedback (PRs #18, #20)

- **Tool:** Claude (claude.ai web, Opus 4.7) for the textual updates
  and diagram redesign; Claude Code (local, WSL) for PR-split strategy,
  branch / commit / PR drafting, and `gh` operations
- **Scope:** `platform` — `docs/architecture/logical_architecture.drawio`
  (+ PNG export), new `docs/architecture/README.md`, and the three
  `docs/claude/` documents (`architecture-decisions.md`,
  `repository-layout.md`, `tooling-reference.md`)
- **What:** addressed the four lecturer-feedback points on the diagram
  (Traefik instead of ingress-nginx, corrected ArgoCD ↔ Crossplane ↔
  Tenants arrows, backend exposed via ingress alongside the frontend,
  monitoring + persistent storage visible), and the architectural
  refinements that came out of the same feedback round (Crossplane
  `provider-helm` replacing ApplicationSets for tenant app deployment,
  central `values/app-version.yaml` + `imageTagOverride` staging,
  GKE Cluster Autoscaler 3–6, `APP_PASSWORD` auth contract,
  CloudNativePG-managed DB credentials, `app-chart` OCI artefact).
- **Verification:** Claude Code diffed against issue #16's acceptance
  criteria, flagged the additional architectural changes beyond the
  issue's scope, proposed a 2-PR split (#18 = diagram + Ingress ADR
  closing #16; #20 = textual refinements closing follow-up issue #19),
  and authored the PR bodies + commit messages.
- **Outcome:** merged — PR #18 (diagram + Ingress ADR, closes #16) and
  PR #20 (textual refinements, closes #19).

## 2026-05-19 — Application stack and deployment plan (issue #21)

- **Tool:** Claude (claude.ai web, Opus 4.7) for drafting; Claude Code
  (local, WSL) for template-compliance review and `gh issue create`
- **Scope:** `platform` — issue #21 body
- **What:** drafted the concrete application proposal — story (property
  management for landlords), stack choices (Node/Fastify/Drizzle,
  React/Vite/Tailwind, JWT-in-cookie auth), Helm chart layout in
  `app-backend/chart/`, six-phase execution sequence, sign-off
  requirements.
- **Verification:** reviewed against the deployment contract in
  `architecture-decisions.md`, removed one dead reference
  (`000-assignment_en.md`), verified `[TASK]` template compliance
  (Context / Scope / Acceptance Criteria / Grading Pillar).
- **Outcome:** open as issue #21, pending team sign-off before phase 1.

## 2026-05-12 — Logical infrastructure diagram

- **Tool:** Claude (claude.ai web, Opus 4.7)
- **Scope:** `platform` — `docs/architecture/logical_architecture.drawio`,
  `docs/architecture/logical_architecture.drawio.png`
- **What:** co-designed the logical infrastructure diagram (GCP project,
  VPC, GKE, Cloud DNS, Secret Manager, GCS, Workload Identity, GHCR,
  external endpoints) and produced an initial draw.io export.
- **Verification:** layout, labels, and component relationships were
  reviewed against `architecture-decisions.md` and `tooling-reference.md`,
  then manually adjusted and polished in draw.io.
- **Outcome:** merged via PR #9

## 2026-05-10 — Initial Claude context bundle

- **Tool:** Claude (claude.ai web, Opus 4.7) for drafting; Claude Code
  (local, WSL) for verifying that the loaded context behaves as intended
- **Scope:** `platform` — `docs/claude/`, `docs/setup/`, `README.md`,
  this file
- **What:** drafted the six context documents (`project-overview.md`,
  `architecture-decisions.md`, `repository-layout.md`, `conventions.md`,
  `tooling-reference.md`, `working-with-claude.md`), the workspace setup
  guide and Claude Code template under `docs/setup/`, the README link,
  and this `AI_USAGE.md` skeleton with first entry.
- **Verification:** Workspace context loader tested live in Claude Code:
- a fresh session was forced through the
  bootstrap reading and probed with questions on tool choices, commit
  format, AI-attribution refusal, and language separation. All checks
  passed.
- **Outcome:** merged via PR (link in PR body)
