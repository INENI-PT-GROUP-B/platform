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
