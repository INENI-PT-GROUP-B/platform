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
