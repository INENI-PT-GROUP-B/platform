# Contributing Guidelines

This document defines the **commit** and **pull request** rules for all repositories
in this organization. Every team member is expected to follow these rules to ensure
a clean, auditable, and consistent project history.

> These rules apply to **all repositories** in the organization.

---

## Table of Contents

- [General Principles](#general-principles)
- [Branching](#branching)
- [Commit Messages](#commit-messages)
- [Pull Requests](#pull-requests)
- [Creating Issues](#creating-issues)
- [Issue Linking](#issue-linking)
- [Reviews & Merging](#reviews--merging)
- [Secrets & Security](#secrets--security)
- [AI Usage Disclosure](#ai-usage-disclosure)

---

## General Principles

1. **No direct commits to `main`** – all changes go through Pull Requests.
2. **No merge commits** – use rebase or squash only (linear history).
3. **Every commit references an issue** – traceability is mandatory.
4. **Small, focused PRs** – easier to review, less risk of conflicts.
5. **No plaintext secrets** – ever. Period.

---

## Branching

### Branch Naming Convention

Use the following pattern:

<initials>/<type>/<short-description>

**Examples:**
- mr/feat/gke-cluster-module
- pp/fix/argocd-sync-loop
- am/docs/update-readme
- rl/ci/add-tflint-workflow

### Branching Rules

- Always branch off `main`
- Keep branches **short-lived** (merge within a few days)
- Delete branches after merge
- Rebase onto `main` if your branch falls behind

```bash
git checkout main
git pull
git checkout -b initials/feat/my-feature
```

## Commit Messages

We follow the Conventional Commits specification.

### Format
<type>(<scope>): <subject>

### Allowed Types

| Type       | Use Case                                       |
|------------|------------------------------------------------|
| `feat`     | New feature                                    |
| `fix`      | Bug fix                                        |
| `docs`     | Documentation only                             |
| `ci`       | CI/CD pipeline changes                         |
| `chore`    | Maintenance, dependencies                      |
| `refactor` | Code refactoring without functional changes    |
| `test`     | Adding or modifying tests                      |
| `perf`     | Performance improvements                       |
| `build`    | Build system or external dependencies          |
| `revert`   | Revert a previous commit                       |

### Common Scopes
iac, gitops, crossplane, argocd, dns, tls, secrets, ci, docs, backend, frontend, workflows

### Subject Rules
- Use the imperative mood: "add feature" not "added feature"
- Lowercase start, no period at the end
- Maximum 72 characters
- Be specific and descriptive

**Examples**
✅ Good:
feat(iac): add GKE cluster module with workload identity
fix(gitops): correct ArgoCD application sync path
docs(readme): add tenant onboarding instructions
ci(workflows): add reusable PR title validation
chore(deps): bump terraform-google to 5.10.0

❌ Bad:
update stuff                    # no type, vague
feat: Added new feature.        # past tense, capitalized, period
fix: bug                        # not descriptive
WIP                             # no type, no info
Merge branch 'main' into feat   # merge commit – forbidden

## Pull Requests

### PR Title
The PR title must follow Conventional Commits – it becomes the squash commit message.

✅ feat(crossplane): add tenant XRD with database composition
❌ Tenant stuff

### PR Description
Use the provided PR template. It must include:

- Summary – what does this PR do?
- Related Issue – Closes #<nr> or Refs #<nr>
- Type of change – feat, fix, docs, etc.
- Changes – bullet list of key modifications
- Checklist – all items checked
- Testing – how was it tested?

### PR Size
- Aim for < 400 lines changed per PR (excluding generated files / lockfiles)
- If larger: split into multiple PRs (e.g. preparation → implementation → docs)

### PR Lifecycle
1. Create branch          →  feat/my-thing
2. Commit & push          →  follow Conventional Commits
3. Open PR                →  fill template, link issue
4. CI runs                →  must be green
5. Request review         →  at least 1 reviewer
6. Address feedback       →  push more commits
7. Squash & merge         →  PR title becomes commit
8. Branch auto-deleted    →  done ✅

## Creating Issues

Every issue uses the org-wide template defined in `INENI-PT-GROUP-B/.github`
at `.github/ISSUE_TEMPLATE/task.yml`. The template is auto-applied in the
GitHub web UI; when using `gh issue create` on the CLI the same structure
must be reproduced manually.

### Required structure

| Element        | Value                                                          |
|----------------|----------------------------------------------------------------|
| Title prefix   | `[TASK]: `                                                     |
| Label          | `task`                                                         |
| Required body  | `Context`, `Scope`, `Acceptance Criteria`, `Grading Pillar`    |
| Optional body  | `References`                                                   |

Each body section is introduced by a `###` heading.

### Grading Pillar values

Pick exactly one:

- `Documentation & Software Management Hygiene`
- `Infrastructure Bootstrap`
- `Application Management`
- `Presentation`
- `Bonus`

### Creating issues via `gh` CLI

The form template only applies in the web UI. When using `gh issue create`,
reproduce all four required sections in the body and pass `--label task`
explicitly:

```bash
gh issue create \
  -R INENI-PT-GROUP-B/<repo> \
  --label task \
  --title '[TASK]: <short imperative summary>' \
  --body "$(cat <<'EOF'
### Context
...

### Scope
...

### Acceptance Criteria
- [ ] ...

### Grading Pillar
<one of the values above>
EOF
)"
```

### Body formatting

Issue, PR, and comment bodies are rendered with GitHub Flavored Markdown.
Unlike `.md` files committed to a repo (which follow CommonMark and reflow
single newlines into spaces), a single newline **inside a paragraph** is
rendered as a hard line break (`<br>`). Do **not** hard-wrap prose
paragraphs in a body: write each paragraph as one logical line and separate
paragraphs with a blank line. Otherwise the text renders with artificial
narrow line breaks regardless of viewport width. Lists, checkboxes, code
fences, and headings are block elements and stay one line per element as
usual.

## Issue Linking
Every PR must reference an issue.

In the PR description:
**Related Issue**
Closes #12

### Keywords
| Keyword                       | Effect                          |
|-------------------------------|---------------------------------|
| `Closes #12`, `Fixes #12`     | Closes issue when PR is merged  |
| `Refs #12`, `Related to #12`  | Links without closing           |

### Cross-Repository References
Closes <org>/<repo>#42
Refs <org>/.github#1

## Reviews & Merging
### Required Checks

Before merging, the following must pass:

- At least 1 approving review
- All CI checks green (PR title validation, commitlint, linters)
- All conversations resolved
- Branch is up to date with main

### Merge Strategy
We use Squash and Merge only.
This ensures:

- Linear history
- One commit per PR on main
- PR title becomes the commit message (must be Conventional Commits!)
- No merge commits

## Secrets & Security

**NEVER commit:**
- API keys, tokens, passwords
- .env files with real values
- Service account JSON keys
- Private keys (.pem, .key)
- Database connection strings with credentials

**If you accidentally commit a secret:**
- Immediately rotate the secret at the source
- Notify the team in the project chat
- Do not just git rm – the secret is in git history
- Coordinate history rewrite (if necessary) with the team

**What to use instead:**
- External Secrets Operator (ESO) for Kubernetes secrets
- Workload Identity for GCP authentication (no JSON keys!)
- OIDC for GitHub Actions → Cloud auth
- .env.example files with placeholder values

## AI Usage Disclosure

We use Generative AI tools (Claude) actively in our daily work. This includes code generation, refactoring, documentation, reviews, and infrastructure design.

### Our Approach

- **AI is a tool, not an authority.** Every AI-generated output is reviewed, understood, and validated by a human before it lands in `main`.
- **You own what you commit.** If an AI helped you write it, you are still responsible for correctness, security, and fit.

### What Not to Do

- Do not paste secrets, customer data, or non-public business information into AI tools
- Do not merge AI-generated code directly (without review)


**All repositories are public (beside repo for app frontend). Assume everything you commit is world-readable.**