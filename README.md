# platform

Central repo for organization-wide working agreements, contribution rules, and shared documentation.

## Contributing

The CONTRIBUTING.md in this repo is the **canonical version** for the entire organization. 
See [CONTRIBUTING.md](./CONTRIBUTING.md).

Key points:
- No direct commits to `main`, all changes via Pull Request
- Every PR references an issue

## Security

This repository is **public**. Do not commit secrets, even in examples. Use placeholder values.

## Cost Planning

GCP cost estimate and credit request basis: [docs/cost-planning](./docs/cost-planning/)

Estimate date: 2026-05-03. Files are static snapshots, future updates land alongside as new dated files.

At project end, the actual spend will be compared against the granted budget and a short lessons-learned note will be added alongside.

## Local Setup & AI Context

This project actively uses generative AI (primarily Claude) for
infrastructure code, documentation, and review. To set up your local
workspace and load the shared project context into your Claude Code
sessions, see [docs/setup/](./docs/setup/README.md).

The shared context bundle for AI-assisted work lives in
[docs/claude/](./docs/claude/) and is loaded automatically when your
workspace is configured per the setup guide.