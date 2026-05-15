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

## 2026-05-15 — Lecturer review feedback integration

- **Contributor:** am
- **Tool:** Claude (claude.ai web, Opus 4.7)
- **Scope:** workspace planning files (`SPRINT-PLAN.md`, `ISSUE-PLAN.md`);
  follow-up edits to `platform/docs/claude/*.md` and `platform/README.md`
- **What:** Processed written feedback from the lecturer on the
  architecture: ingress-nginx → Traefik (upstream deprecation in 2025);
  persistent-storage strategy made explicit (`standard-rwo`, no custom
  StorageClass); Argo CD ↔ Crossplane arrow direction corrected in the
  planned system diagram; visibility for the storage and monitoring layers
  required in the diagram. Claude flagged that one of the feedback points
  (Ingress → Frontend → Backend topology) was based on the pre-pivot
  architecture and drafted reply wording that communicates the pivot back
  to the lecturer. While processing the feedback, Claude identified that
  the canonical architecture documentation in `platform/docs/claude/` had
  drifted from the workspace plans (Gitea pivot not yet propagated to
  ADR / repo layout / tooling reference) and proposed targeted file-level
  edits to bring them in sync.
- **Verification:** Each feedback point cross-referenced to a concrete
  plan edit. Audit script re-run after edits — issue-balance preserved at
  15/15/15/15 across all four pillars.
- **Outcome:** Plans updated in workspace (SPRINT-PLAN Rev 4, ISSUE-PLAN
  Rev 6). Canonical-doc updates (this PR series) bring
  `architecture-decisions.md`, `repository-layout.md`,
  `tooling-reference.md`, and `project-overview.md` to the post-pivot
  state.

## 2026-05-13 — Issue-plan balance rework

- **Contributor:** am
- **Tool:** Claude (claude.ai web, Opus 4.7)
- **Scope:** workspace planning file `ISSUE-PLAN.md` (Rev 3 → Rev 5)
- **What:** Iterated three times over the issue distribution to reach
  exact 15/15/15/15 author and reviewer balance per grading pillar,
  without dropping any deliverable. First attempt dropped a Bootstrap
  issue (per-tenant Grafana dashboard) — Claude was correctly pushed back
  on for shrinking scope. Final approach: merged the two halves of the
  Argo CD self-management chain (`bootstrap-app.yaml` in `platform-iac`
  and `applications/root.yaml` in `platform-gitops`) into one tracking
  issue spanning two repositories. Both files are still authored; two
  PRs (one per repo) reference the single merged issue. Claude wrote a
  Python audit script that parses the markdown table and verifies
  per-member per-pillar counts plus per-sprint reviewer distribution.
- **Verification:** Audit script run after each revision. Per-sprint
  reviewer table cross-checked against the §6 summary by recomputing
  from the row data; corrected a stale ±1 drift in the prior revision's
  §6 numbers.
- **Outcome:** `ISSUE-PLAN.md` Rev 5 in workspace; later refined to
  Rev 6 with the lecturer-feedback session (entry above).

## 2026-05-13 — Application pivot to Gitea and bootstrap-script approach

- **Contributor:** am
- **Tool:** Claude (claude.ai web, Opus 4.7)
- **Scope:** workspace planning files `SPRINT-PLAN.md` and `ISSUE-PLAN.md`;
  architectural impact across all repos
- **What:** Discussed switching the tenant application from a
  custom-authored backend + frontend (forked from the lecturer's reference
  app, planned to be replaced with our own minimal stack) to **Gitea**,
  deployed via the upstream Helm chart at
  `oci://docker.gitea.com/charts/gitea`. Claude worked through the
  trade-offs: off-the-shelf vs handcrafted application; demonstrable
  multi-tenancy story; image-build pipeline scope; effort reallocation
  across the four grading pillars. Pivot accepted. As part of the same
  session, decided to replace the previously-planned custom-application
  CI/CD build pipeline with a single Day-1 bootstrap script
  (`platform/scripts/bootstrap.sh`) for one-click platform deployment.
  Rationale: a script is auditable in one place, runnable from a developer
  machine without CI access to long-lived state, and keeps Day 1 entirely
  outside the application-deployment surface. Both planning documents
  fully revised; per-pillar contribution balance preserved.
- **Verification:** Plans cross-checked against `project-overview.md`
  hard constraints. At the time of the session,
  `architecture-decisions.md` was not yet updated for the pivot — the
  drift was identified later (see 2026-05-15 entry) and is being
  resolved in the current canonical-doc PR series.
- **Outcome:** `SPRINT-PLAN.md` Rev 2 and `ISSUE-PLAN.md` Rev 3 in
  workspace. Subsequent revisions refined balance (Rev 4–6) and
  integrated lecturer feedback.

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
