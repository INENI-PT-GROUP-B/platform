# Working with Claude

> How this team uses Claude (and other generative AI) effectively and
> safely. Companion document to the AI Usage Disclosure section in
> [`CONTRIBUTING.md`](../../CONTRIBUTING.md). The CONTRIBUTING document is
> the canonical policy; this document is the operational guide.

## Loading context into a session

When you start a Claude session for project work, share the relevant
files from this folder. Minimum useful set:

- [`project-overview.md`](./project-overview.md) — always
- [`architecture-decisions.md`](./architecture-decisions.md) — almost always
- [`repository-layout.md`](./repository-layout.md) — when working across repos
- [`conventions.md`](./conventions.md) — when authoring commits or PRs
- [`tooling-reference.md`](./tooling-reference.md) — when generating manifests, IaC, or tool-specific code
- [`working-with-claude.md`](./working-with-claude.md) — this file, for the rules below

If you use Claude Code or Claude Desktop, point it at the
`platform/docs/claude/` folder; the files load as context.

## What Claude must NOT do

These are absolute rules:

- **No AI attribution in git history.** Never add `Co-Authored-By: Claude
  <noreply@anthropic.com>` trailers, never add "🤖 Generated with Claude
  Code" or similar footers to commits or PR descriptions, never add
  attribution comments inside code.
- **No mixed-language artefacts.** Everything we produce — code, commits,
  PRs, issues, docs, comments — is in English.
- **No secrets, customer data, or non-public business information** pasted
  into AI tools, ever. (We have none of these in this academic project,
  but the rule applies regardless.)
- **No direct merging of AI-generated code.** Every output is reviewed
  and validated by a human before it lands in `main`.
- **No fabricated facts.** If Claude is uncertain about a tool version, a
  GCP API field, or a Kubernetes resource shape, it must say so and
  suggest verification — not invent.

## What Claude is good for

- Generating boilerplate (Helm values, Terraform modules, Kubernetes
  manifests, GitHub Actions workflows)
- Writing draft documentation and refining wording
- Reviewing PRs for obvious issues, missing tests, security concerns
- Explaining errors from `terraform plan`, `kubectl describe`, Argo CD
  sync failures
- Comparing options ("Cloud SQL vs CloudNativePG", "ArgoCD ApplicationSet
  generators")
- Generating commit messages and PR descriptions in Conventional Commits
  format

## What Claude is bad at (verify before trusting)

- Specific tool versions and API field names — these change often
- GCP IAM role names and least-privilege scoping
- Crossplane patch syntax across versions — verify against current docs
- "Latest best practice" claims — Claude's training data is bounded;
  always check official docs for current recommendations

## Authorship and ownership

The author of every commit is the human team member who reviewed and
submitted it. AI is not added as Co-Author. If Claude wrote 100% of a
file, the human who reviewed and committed it owns the result and is
accountable for its correctness.

## AI usage logging

Significant uses of AI are logged in
[`AI_USAGE.md`](../../AI_USAGE.md) at the root of the `platform` repo.

A "significant use" is something worth showing the lecturer at the demo:

- Generating a substantial component (an entire Terraform module, the
  initial app code, a complete Crossplane Composition)
- Architecture or trade-off decisions where AI helped reason through
  alternatives
- Notable troubleshooting sessions where AI accelerated diagnosis

Trivial uses (autocompleting a function name, polishing a sentence in a
comment) do not need to be logged.

## Recommended user-level config (Claude Code)

For team members using Claude Code locally, add to your user-level memory
(`~/.claude/CLAUDE.md` on Linux/WSL, `%USERPROFILE%\.claude\CLAUDE.md` on
Windows):

```text
NEVER add "Co-Authored-By: Claude" trailers or "Generated with Claude Code"
footers to commits or pull requests. NEVER add AI attribution comments to
code. Output language is English. When working in the INENI-PT-GROUP-B
project, read platform/docs/claude/*.md for project context.
```

This is a backstop — Claude Code occasionally tries to add attribution
footers despite project-level instructions. The user-level memory loads
in every session.

## Updating these files

Changes to any file in this folder go through a normal PR in the
`platform` repository. After merge, team members can `git pull` the
`platform` repo to refresh their local copy.

If a Claude session reveals that a context file is out of date or
misleading, open an issue in the `platform` repo describing what needs
updating.
