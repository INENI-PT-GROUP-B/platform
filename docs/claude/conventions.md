# Conventions Cheat-Sheet

> **Single source of truth: [`CONTRIBUTING.md`](../../CONTRIBUTING.md).**
> This file is a quick reference for AI-assisted work. If anything here
> conflicts with `CONTRIBUTING.md`, the canonical document wins.
>
> When in doubt — or when working on something not covered here — read
> `CONTRIBUTING.md` directly.

## Branching

Branch from `main`, name as `<initials>/<type>/<short-description>`.

| Initials | Member  |
| -------- | ------- |
| `mr`     | Marco   |
| `am`     | Alex    |
| `rl`     | Ronny   |
| `pp`     | Patrick |

Examples:
- `rl/feat/gke-cluster-module`
- `am/fix/argocd-sync-loop`
- `pp/docs/update-readme`
- `mr/ci/add-tflint-workflow`

Branches are short-lived and deleted after merge.

## Commit messages

Format: `<type>(<scope>): <subject>`

Allowed types (must match `.commitlintrc.json`):

```
feat | fix | docs | ci | chore | refactor | test | perf | build | revert
```

Common scopes (not exhaustive):

```
iac | gitops | crossplane | argocd | dns | tls | secrets
ci  | docs   | backend    | frontend | workflows
```

Subject rules:
- imperative mood ("add", not "added")
- lowercase start, no trailing period
- max 72 characters
- specific and descriptive

Examples:
- ✅ `feat(iac): add GKE cluster module with workload identity`
- ✅ `fix(gitops): correct ArgoCD application sync path`
- ❌ `update stuff` — no type, vague
- ❌ `feat: Added new feature.` — past tense, capitalized, period
- ❌ `WIP` — no type, no info

Every commit references an issue (in body or via PR).

## Pull requests

- PR title follows Conventional Commits (becomes the squash commit message)
- PR description uses the org PR template (Summary, Related Issue, Type,
  Changes, Checklist, Testing)
- Link an issue with `Closes #<n>` or `Refs #<n>`; cross-repo with
  `Closes INENI-PT-GROUP-B/<repo>#<n>`
- Aim for under 400 lines changed per PR (excluding lockfiles, generated
  files)
- Squash and merge only — no merge commits, linear history
- At least one approving review and all CI checks green before merge

## Required CI checks (enforced via `.github` reusable workflows)

- `commitlint` — validates each commit message in the PR
- `pr-title` — validates the PR title (becomes the squash commit message)
- Repo-specific linters as configured (e.g. `tflint`, `yamllint`,
  `kubeconform`, `helm lint`)

## What not to commit

Never:
- API keys, tokens, passwords
- `.env` files with real values
- service account JSON keys
- private keys (`.pem`, `.key`)
- database connection strings with credentials
- AI attribution (no `Co-Authored-By: Claude`, no boilerplate footers,
  no AI mention in code comments)

If a secret was accidentally committed:
1. rotate the secret at the source immediately
2. notify the team
3. coordinate history rewrite if needed — `git rm` alone is not enough

## Language

All code, commits, PR descriptions, issues, comments, and documentation
are written in **English**.

## Cross-repository references

Use `<owner>/<repo>#<number>`:

```
fix(crossplane): handle DB connection retry on tenant create

Refs INENI-PT-GROUP-B/platform-gitops#42
Closes INENI-PT-GROUP-B/platform#7
```
