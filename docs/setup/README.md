# Local Workspace Setup

This folder contains setup files that team members copy into their **local
workspace**. Files here are not consumed by tools running in the cloud — they
are templates for personal developer environments.

## What is here

- `workspace-CLAUDE.md.template` — a context-loader file for Claude Code.
  When placed at the root of your local workspace, it tells Claude Code where
  to find the project's context documents in `platform/docs/claude/`.

## Why this exists

We want every team member's Claude Code session to load the same project
context: architecture decisions, conventions, repository layout, tooling
patterns. The canonical files live in
[`platform/docs/claude/`](../claude/) and are committed to the
`platform` repository.

To make Claude Code pick them up automatically — regardless of whether you
start your session in `platform/`, `platform-iac/`, `platform-gitops/`, or
your workspace root — we use a single workspace-level `CLAUDE.md` that
points to those files. Claude Code searches upward from the current
directory until it finds a `CLAUDE.md`, so a workspace-level file works
from any sibling repository.

## Workspace structure assumed by the template

The template assumes you clone all six repositories as **siblings** in one
parent directory. Pick any name for the parent — the template uses
relative paths and does not care.

```
<your-workspace-root>/             ← e.g. ~/dev/ineni or C:\code\ineni
├── CLAUDE.md                      ← copy of workspace-CLAUDE.md.template
├── platform/
├── platform-iac/
├── platform-gitops/
├── app-backend/
├── app-frontend/
└── .github/
```

If your layout is different (e.g. nested under another folder), you must
adjust the paths inside the copied `CLAUDE.md` accordingly.

## Setup — WSL or Linux

```bash
# Adjust the path to your workspace root
cd ~/dev/ineni

# Copy the template, dropping the .template extension
cp platform/docs/setup/workspace-CLAUDE.md.template CLAUDE.md

# Verify Claude Code picks it up — start a session and ask it to list its context
cd ~/dev/ineni
claude
```

Inside the Claude session:

```
What CLAUDE.md files do you have loaded? List them by path.
```

You should see your `~/dev/ineni/CLAUDE.md` listed alongside your
user-level `~/.claude/CLAUDE.md` (if you have one).

## Setup — Windows (PowerShell)

```powershell
# Adjust the path to your workspace root
cd C:\code\ineni

# Copy the template, dropping the .template extension
Copy-Item platform\docs\setup\workspace-CLAUDE.md.template CLAUDE.md

# Verify
cd C:\code\ineni
claude
```

## Updating

The template is committed and versioned. When it changes:

1. Pull the latest `platform` repository: `cd platform && git pull`
2. Re-copy the template to your workspace root (overwriting your local
   `CLAUDE.md`), or merge changes manually if you have local
   customizations.

## What this is NOT

- It is **not** a replacement for the project context documents in
  `platform/docs/claude/`. Those are the actual content; this template is
  just a pointer.
- It is **not** a Claude Code user-level memory. Per-user rules (preferred
  language, OS-specific paths, personal preferences) belong in
  `~/.claude/CLAUDE.md` (Linux/WSL) or `%USERPROFILE%\.claude\CLAUDE.md`
  (Windows).
- It is **not** committed at the workspace root, because the workspace
  root is not a git repository. Each team member has their own copy
  locally.

## See also

- [`platform/docs/claude/working-with-claude.md`](../claude/working-with-claude.md)
  — operational guide for AI-assisted work
- [`platform/CONTRIBUTING.md`](../../CONTRIBUTING.md) — canonical
  conventions for the organization
