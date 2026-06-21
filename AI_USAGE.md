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

## 2026-06-21 — Consolidated catch-up: app hardening, CI modernization, tenant follow-ups, docs (rl)

- **Tool:** Claude Code (local, WSL/Debian, Opus 4.7)
- **Scope:** several smaller AI-assisted changes from 2026-06-08 to
  2026-06-20, logged together rather than per PR:
  - `app-frontend` — nginx runtime hardening (#22), version badge (#21),
    Node-24 action bump (#23).
  - `app-backend` — graceful shutdown (#26), validation/docs alignment
    (#27), Node-24 action bump (#28).
  - `.github` — semantic-pull-request action v6 bump (#13), organization
    profile README (#15).
  - `platform-gitops` — S3-09 tenant Composition follow-ups: BasicAuth
    plaintext to GSM (#79), tenant egress to the API server (#78),
    validation evidence (#82).
  - `platform-iac` — provider-gcp Secret Manager IAM broadening (#60).
- **What:** Claude assisted across this batch at a drafting / diagnosis
  level — the frontend nginx hardening and version badge, the backend
  graceful-shutdown handler, the Node-24 / semantic-pull-request CI sweep,
  the S3-09 tenant follow-ups and the matching IAM broadening, and the
  organization profile README. In each case the design, scoping, and final
  review were the team member's; the AI output was a draft.
- **Verification:** each change passed the repo CI gates (lint/build/test,
  yamllint/kubeconform/helm lint, commitlint, pr-title, as applicable) and
  a human review before merge; cluster-touching changes were checked live
  where relevant.
- **Outcome:** merged — `app-frontend` #21/#22/#23, `app-backend`
  #26/#27/#28, `.github` #13/#15, `platform-gitops` #78/#79/#82,
  `platform-iac` #60.

## 2026-06-19 — S4-05 cost actuals comparison (pp)

- **Tool:** Claude Code (local, macOS, Opus 4.7)
- **Scope:**
  - `INENI-PT-GROUP-B/platform`#121 — new
    `docs/cost-planning/actuals_costs_gcp.md` sibling to the S1-15
    `capacity_costs_gcp.md` estimate, plus the source data folder
    `docs/cost-planning/actuals/` (`Closes platform#100`).
  - `platform` — this entry.
- **What:** S4-05 produced the actuals-vs-budget comparison for the
  Sprint 1-4 active window. Claude was used in a supporting role on
  the comparison itself: matching each service line in the GCP
  billing CSV to the corresponding S1-15 estimate line, normalising
  cluster-active services to a per-active-month rate (cost × 30 / 18
  for the ~18 cluster-active days in the 40-day window), and
  surfacing the two drivers of the credited-window −18 % delta —
  GKE control-plane fee absorbed by "Always Free" on the positive
  side, Cloud Monitoring via Managed Prometheus on the negative
  side. The K8s Engine credit pattern took several rounds to read
  correctly: the daily billing chart shows the management-fee bars
  step up on 2026-06-14, which is before the S4-04b cluster
  redeploy. Claude helped pin the cause to the shared Always Free
  cap on the HAW educational billing account and to derive the
  steady-state rate from 2026-06-14 onwards (~430 €/month — ~3 %
  under the estimate, ~14 % under the granted budget).
- **Verification:** Every figure in the comparison table was
  cross-checked against the CSV that ships in the PR; the
  2026-06-14 credit-exhaustion date was read off the daily billing
  chart in the source PDF; the steady-state per-month rate was
  re-derived by hand from the per-service rows. Repo CI
  (markdownlint, commitlint, pr-title) validated the doc shape.
- **Outcome:** merged (#121, 2026-06-19). Closes `platform#100`.

## 2026-06-15 — S4-04b cluster redeploy validation (pp)

- **Tool:** Claude Code (local, macOS, Opus 4.7)
- **Scope:**
  - `INENI-PT-GROUP-B/platform-gitops`#108 — new
    `docs/cluster-redeploy-validation.md` with the live evidence
    (S4-04b, `Closes platform-gitops#65`).
  - `INENI-PT-GROUP-B/platform-iac`#68 — `TEARDOWN.md` aligned with
    the procedure the live run actually needed.
  - `INENI-PT-GROUP-B/platform-iac`#69 — `force_destroy = true` on
    the pg-backups bucket (`Closes platform-iac#67`).
  - `INENI-PT-GROUP-B/platform-gitops`#109 —
    `refreshInterval: 5m` on the per-tenant BasicAuth
    ExternalSecret to narrow the ESO/SecretVersion race window
    (`Closes platform-gitops#107`).
  - Follow-up issues opened during the write-up:
    `platform-iac#66` (broken IAM-preflight on gcloud SDK 572.0.0,
    handed to mr), `platform-iac#67`, `platform-gitops#107`.
  - `platform` — this entry.
- **What:** S4-04b ran as a full teardown + rebootstrap against
  `dotted-axle-495612-f4`. Claude assisted with the cluster-query
  sequence at each phase — which `kubectl get` / `gcloud` to fire,
  in which order, with which `jsonpath` — and with structuring the
  evidence doc against the existing `multi-tenancy-validation.md` /
  `lifecycle-tests.md` patterns. The live run surfaced eight gaps
  between TEARDOWN.md as written and what an operator actually
  needs today, plus one incorrect assumption in Issue #65's own
  scope: pg-backups was listed as persistent there, but
  `terraform state list` showed
  `module.backup.google_storage_bucket.pg_backups` in the destroy
  plan and the live destroy confirmed it. The cluster-redeploy doc
  records the correction and `#67` makes the destroy actually
  succeed on a populated bucket. The trickiest one to diagnose was
  the ESO/SecretVersion race on the per-tenant BasicAuth: the
  operator-visible plaintext in GSM looked like it should work, but
  Traefik kept accepting only the old password. Claude helped
  cross-check the GSM htpasswd version timestamps against the K8s
  Secret content; the race became obvious once the three were lined
  up, and `#109` narrows the catch-up window from 60 min to 5 min.
  The `dependsOn` / `forceSync` options that would eliminate the
  window entirely are tracked in `#107` for a follow-up.
- **Verification:** Every cluster + GCP output in the docs was
  pulled live with `kubectl` or `gcloud` and copied verbatim — no
  values were invented or paraphrased. The bucket-persistence
  correction was cross-checked against `terraform state list`
  before `#67` was opened. The BasicAuth race was confirmed against
  three independent signals (the ESO `ExternalSecret` `refreshTime`,
  the Crossplane `SecretVersion` creation timestamp in GSM, and the
  bcrypted hash in the K8s Secret data key — all three agreed).
  Repo CI (yamllint, kubeconform, helm lint, markdownlint,
  commitlint, pr-title, terraform fmt + validate + tflint)
  validated every PR's manifest and doc shapes.
- **Outcome:** `platform-gitops`#108 + #109 open, `platform-iac`#68
  + #69 open, `platform-iac`#66 + #67 + `platform-gitops`#107
  opened as follow-ups (mr taking #66). This entry in `platform`.

## 2026-06-14 — S3-10 rollout verification + S4-04a app crash recovery (pp)

- **Tool:** Claude Code (local, macOS, Opus 4.7)
- **Scope:**
  - `INENI-PT-GROUP-B/platform-gitops`#95 (new issue) + PR
    `platform-gitops`#96 — completing the v0.1.1 lockstep across
    `crossplane/configmaps/app-version-cm.yaml` and the Composition
    Resource 15 `chart.version` field.
  - `INENI-PT-GROUP-B/platform-gitops`#97 — `docs/lifecycle-tests.md`
    with the S4-04a backend pod crash recovery test.
  - `INENI-PT-GROUP-B/platform-gitops`#105 —
    `docs/multi-tenancy-validation.md` Test 5 (S3-10 global app-version
    rollout evidence).
  - `platform` — this entry.
- **What:** S3-10 was approached as a live verification of the rollout
  mechanism. Claude was used to draft the cluster-query sequence and
  structure the doc against the existing `multi-tenancy-validation.md`
  pattern. The very first query — a cross-check of the in-cluster
  `tenant-app-defaults` ConfigMap against `values/app-version.yaml` on
  main — showed an unexpected gap: the central #94 bump had not
  propagated, every tenant pod was still on the pre-bump tags. The
  header comment in `crossplane/configmaps/app-version-cm.yaml`
  documented the 3-file lockstep requirement, #94 had only edited the
  first; the two missed files (the in-cluster ConfigMap content and the
  Composition Resource 15 `chart.version` field) were bumped in #96 and
  the rollout completed live on the next Argo CD sync. The S3-10
  evidence in #105 then captures the subsequent v0.1.1 → v0.1.2
  frontend cycle where the mechanism worked end-to-end. S4-04a (#97)
  was run against demotenant1: a backend pod was killed and three
  independent signals were captured around the delete — the
  `kubectl get pod -w` watcher, the pod-condition timestamps via
  `jsonpath`, and a parallel curl-on-port-forward polling loop. All
  three agreed on a 21 s delete → Ready window. The lifecycle-tests
  file was structured to mirror `multi-tenancy-validation.md` (Setup /
  Procedure / Outputs / Proves).
- **Verification:** Every cluster output in the docs was pulled live
  with `kubectl` and copied verbatim — no values were invented or
  paraphrased. The lockstep gap was confirmed against three sources
  (the live ConfigMap, `values/app-version.yaml` on main, and the
  ConfigMap file header comment) before #96 was opened. The S4-04a
  21 s window was reconstructed from three independent signals and
  cross-checked; all agreed. Repo CI (yamllint, kubeconform, helm lint,
  markdownlint, commitlint, pr-title) validated the manifest and doc
  shapes.
- **Outcome:** `platform-gitops`#96 merged 2026-06-13.
  `platform-gitops`#97 and #105 open. This entry in `platform`.

## 2026-06-14 — Tenant onboarding demo runbook draft (rl)

- **Tool:** Claude Code (local, WSL/Debian, Opus 4.7)
- **Scope:** `INENI-PT-GROUP-B/platform`#113 + PR `platform`#115 — new `docs/presentation/demo-script.md`.
- **What:** drafted the S5-02 reusable demo runbook for end-to-end tenant onboarding (pre-flight, optional pre-staged PR, claim shape, step-by-step demo trail, timing, fallback, cleanup). Cross-links to am's `tenant-onboarding.md` rather than duplicating. Tenant name parametrised as `livedemoNN` placeholder so the GSM identity is fresh per run.
- **Verification:** every embedded `kubectl` command run against the live cluster with `staging` substituted; example outputs match real output verbatim. Composition-derived names cross-checked against the XRD and `xtenant-default.yaml`. Five mid-draft corrections from the live verification round (wrong GKE zone, wrong htpasswd Secret name in the tenant namespace, missing `-n crossplane-system` on `kubectl get tenant`, claim-vs-composite column header swap, overestimated timing total) — all caught before commit.
- **Outcome:** merged — `platform`#115 (2026-06-19), closes `platform`#113.

## 2026-06-13 — Crossplane `string.fmt` constant-name patch bug (rl)

- **Tool:** Claude Code (local, WSL/Debian, Opus 4.7)
- **Scope:** `INENI-PT-GROUP-B/platform-gitops`#103 + PR `platform-gitops`#104 — `imageTagOverride` patches in `crossplane/compositions/xtenant-default.yaml`.
- **What:** the v0.1.2 rollout surfaced that `imageTagOverride` propagated to the backend image tag but silently dropped on every constant-name patch (frontend tag, Backup `ObjectStore` name). Crossplane's `string.fmt` passes the source value as a `fmt.Sprintf` argument; without a `%` verb the constant string renders unchanged and the input is discarded. Fix: add `%.0s` to every constant-name patch so the input is absorbed.
- **Verification:** the bug shipped in `platform-gitops`#94 and survived the pre-merge AI review because the review was code-only — `helm template` against the value shape, schema checks, no live reconciliation. It only surfaced once the v0.1.2 rollout made the backend/frontend tag divergence end-user-visible. Post-fix verified by re-running the rollout against staging + central with both tags flipping in lockstep.
- **Outcome:** merged — `platform-gitops`#104 (closes #103).

## 2026-06-11 — CNPG NetworkPolicy diagnosis: status probe + failing backups (rl)

- **Tool:** Claude Code (local, WSL/Debian, Fable 5)
- **Scope:**
  - `INENI-PT-GROUP-B/platform-gitops`#91 (new issue) + PR
    `platform-gitops`#92 — Composition resource 7b
    (`netpol-allow-cnpg-operator`) in
    `crossplane/compositions/xtenant-default.yaml`.
  - Status comments on `platform-gitops`#83 (S3-08 acceptance evidence)
    and `platform-gitops`#67 (backup verification heads-up).
- **What:** A routine S3-08 acceptance check surfaced that all four
  tenant CNPG Clusters report `Instance Status Extraction Error: HTTP
  communication issue`. Claude ran the diagnosis end-to-end: repo-side
  rationale check (the tenant NetworkPolicy design never considered
  operator ingress — gap, not decision), upstream verification (CNPG
  networking docs: operator must reach instance pods on TCP 8000/5432),
  and live-cluster evidence (operator logs with `dial tcp <pod>:8000:
  i/o timeout`, Backup object inventory, policy dump). The live data
  refuted the initial hypothesis that the blocked status port prevents
  backups from starting: all 8 backups had started and failed inside
  barman-cloud with `Project was not passed and could not be determined
  from the environment` — a second, independent gap. Claude traced it
  to the egress policy blocking the GKE metadata server that
  `gkeEnvironment: true` ADC depends on, and verified the
  dataplane-specific endpoint against Google's GKE auth troubleshooting
  doc (Dataplane V2: `169.254.169.254/32:80`; V1 would be
  `169.254.169.252/32:988`), matching the ~42s metadata-retry pattern
  in the barman logs. The fix is one additional per-tenant
  NetworkPolicy scoped via a `cnpg.io/cluster` Exists match (no
  per-tenant patch), ingress 8000+5432 from `cnpg-system`, egress TCP
  80 to the metadata IP.
- **Verification:** Pod labels, dataplane (`anetd`,
  `datapath_provider = "ADVANCED_DATAPATH"` pinned in IaC) and the
  `gke-metadata-server` ports checked live before writing the rule;
  `yamllint` clean locally; PR CI green (yamllint, kubeconform, helm
  lint). The behavioural acceptance criteria (Cluster `Ready=True`,
  first base backup + WAL files in the bucket) are live-gated
  post-merge and tracked in `platform-gitops`#91.
- **Outcome:** issue `platform-gitops`#91 and PR `platform-gitops`#92
  open; #83/#67 comments posted; this entry.

## 2026-06-10 — Per-tenant CNPG backups to GCS via direct WIF (S3-04 follow-up) (am)

- **Tool:** Claude Code (local, Windows, Fable 5)
- **Scope:**
  - `INENI-PT-GROUP-B/platform-gitops`#67 — Composition resources 17
    (`BucketIAMMember`) + 18 (`ScheduledBackup`), `backup:` stanza on the
    CNPG `Cluster`, `provider-gcp-storage` install + DRC.
  - `INENI-PT-GROUP-B/platform-iac` — Workload Identity binding for the
    `provider-gcp-storage` KSA on the existing crossplane GSA.
  - `platform` — § Database § Backups in `architecture-decisions.md`,
    family-layout / Option A updates, this entry.
- **What:** Worked the open credential decision in #67 with Claude:
  compared classic Workload Identity (per-tenant GSA + impersonation,
  needs an IAM sub-provider) against WIF direct resource access
  (`principal://` bucket binding, no GSA), and settled on direct WIF.
  Claude verified the load-bearing facts against vendor docs before
  implementation: CNPG `googleCredentials.gkeEnvironment` semantics and
  the in-tree Barman deprecation timeline (operator 1.29.1 via chart
  0.28.2 still ships it), the WIF principal subject format, the Upbound
  `BucketIAMMember` v1beta1 schema incl. `condition`, and the GCS gotcha
  that prefix conditions deny `objects.list` unless paired with the
  `objectListPrefix` API attribute. It then generated the Composition
  resources, provider install, DRC, and the IaC binding.
- **Verification:** `terraform fmt -check` + `terraform validate` locally;
  manifests cross-checked field-by-field against the Upbound marketplace
  schema and CNPG 1.29 docs; repo CI (yamllint, kubeconform, tflint).
  Live backup verification (first base backup + WAL files appearing under
  the tenant prefix) is pending post-merge and tracked in #67.
- **Outcome:** PRs open in `platform-gitops`, `platform-iac`, `platform`

## 2026-06-08 — Sprint 3 close prep: chart fix + multi-tenancy PR1 (S3-09) (rl)

- **Tool:** Claude Code (local, WSL/Debian, Opus 4.7), plus a second
  planning pass on claude.ai (web, Opus 4.8) before starting.
- **Scope:**
  - `INENI-PT-GROUP-B/app-backend`#21 + PR `app-backend`#23 — chart
    Ingress TLS `secretName` typo against `docs/chart-contract.md`.
  - `INENI-PT-GROUP-B/app-backend`#22 — cut `v0.1.0` release.
  - `INENI-PT-GROUP-B/app-frontend`#17 — cut `v0.1.0` release.
  - `INENI-PT-GROUP-B/platform-gitops`#75 — `feat(tenants)`
    demotenant1 + demotenant2 for `platform-gitops`#62 (S3-09).
- **What:** Worked through S3-09 end-to-end with Claude. Plan mode
  in Claude Code for the local work, plus a second planning pass on
  Opus 4.8 in the web before starting. The preflight caught three
  things I would have hit later: the `v0.1.0` tags in
  `values/app-version.yaml` did not exist anywhere yet, the chart's
  TLS `secretName` had drifted from the contract by a missing `-tls`
  suffix, and the `S3-09 → S3-08` dependency in the task list turned
  out to be plan sequencing rather than a real technical gate.
- **Verification:** `helm lint` + `helm template` against the
  Composition's value shape rendered the corrected Ingress.
  Tenant claims yamllint-clean. Live-cluster validation belongs to
  the follow-up doc PR.
- **Outcome:** Three issues opened, two PRs opened
  (`app-backend`#23, `platform-gitops`#75).

---

## 2026-06-06 — S3-05 BasicAuth + GHCR-pull Composition extension (pp)

- **Tool:** Claude Code (local, macOS, Opus 4.7)
- **Scope:** `INENI-PT-GROUP-B/platform-gitops`#58 — the S3-05 extension of the
  shared `xtenant-default` Composition with seven new resources: ESO `Password`
  generator + bcrypt-htpasswd `ExternalSecret` in `crossplane-system`, Crossplane
  `Secret` + `SecretVersion` under `secretmanager.gcp.upbound.io` writing to
  Google Secret Manager, two tenant-namespace `ExternalSecret`s (htpasswd
  materialisation + GHCR dockerconfigjson), and a Traefik BasicAuth
  `Middleware`. The Composition grew from 401 to 743 LOC.
- **What:** The plan on `platform-gitops`#39 was iterated three times before
  the implementation. The first sketch used an ESO `PushSecret` for the GSM
  write; rejected because it would have left `provider-gcp-secretmanager`
  installed but unused, breaking the IAM split from S1-08 where the
  `external-secrets` GSA is read-only and the `crossplane-provider-gcp` GSA
  holds write. The second sketch used a Kubernetes `Job` with a custom image
  to bcrypt-hash; rejected as unnecessary plumbing. The shipped design is
  hybrid: ESO generates and bcrypt-hashes via the Sprig `htpasswd` template,
  Crossplane provider-gcp-secretmanager writes the `SecretVersion` via
  `secretDataSecretRef` consuming the in-cluster Secret.
- **Verification:** A pre-push verification round cross-checked each API field
  against upstream sources. Three substantive bugs were caught before the
  first push: the API subdomain was `secret.gcp.upbound.io` in the local draft
  but `secretmanager.gcp.upbound.io` in the v2.5.0 provider examples (the ADR
  carried the same wrong subdomain — addressed by the follow-up doc-sync); the
  v1beta1 replication form was spelled `automatic` instead of the correct
  `auto`; the Sprig `htpasswd` template was called with two arguments while
  the vendored sprig in ESO v2.5.0 takes three (`bcrypt` algorithm). Each fix
  was followed by a `ruby -ryaml -e 'YAML.load(...)'` re-parse against the
  composition file. Real end-to-end reconciliation against the live cluster
  is gated on S3-08 (am, staging tenant).
- **Outcome:** Merged — `platform-gitops`#58 at 12:05 UTC, auto-closed
  `platform-gitops`#39.

## 2026-06-05 — Sprint 3 xtenant-default Composition + Tiering doc (S3-02, S3-03) (rl)

- **Tool:** Claude Code (local, WSL/Debian, Opus 4.7)
- **Scope:** `INENI-PT-GROUP-B/platform`#92 (Tiering paragraph in
  `docs/claude/architecture-decisions.md` § Multi-tenancy) and
  `INENI-PT-GROUP-B/platform-gitops`#55 (the first
  `crossplane/compositions/xtenant-default.yaml`). The Composition PR closes both
  S3-02 (`platform-gitops`#38, reassigned offline from pp) and S3-03
  (`platform-gitops`#54).
- **What:** Two artefacts in sequence. (1) Tiering paragraph — Claude proposed
  three framing options (Headroom-orientiert / SaaS-Pricing / Capability-
  Demonstration); I picked Capability-Demonstration. An am PR review pointed out
  that the XRD-description and `platform-gitops`#38 scope both name ResourceQuota
  + LimitRange, so the paragraph was widened on a follow-up commit and the plan
  adjusted so `LimitRange.max` is tier-mapped while the other LimitRange fields
  stay constant. (2) Composition — 401 LOC Crossplane v1.x P&T rendering seven
  resources via `provider-kubernetes` `Object`: Namespace, ResourceQuota (tier-
  mapped on 7 fields), LimitRange (per-container; `max` tier-mapped), and four
  NetworkPolicies (default-deny ingress, default-deny egress + DNS allow,
  permissive in-namespace 5432 decoupled from S3-04, traefik → app on 80/3000).
  Two commits on the dev branch so the slices were readable in review.
- **Verification:** Plan mode with two `Explore` subagents covering the XRD shape
  + provider-kubernetes install and the cluster footprint + app baseline.
  `yamllint` exit 0 locally on both files. Tier-sizing numbers derived from the
  documented cluster bounds (12–24 vCPU / 48–96 GiB) and the
  `app-backend/chart/values.yaml` pod numbers. Cluster reconciliation lands with
  S3-08 (am, staging tenant).
- **Outcome:** merged — `platform`#92 at 16:00 UTC (auto-closed `platform`#90);
  `platform-gitops`#55 at 18:11 UTC after a rebase onto the `platform-gitops`#57
  fix (auto-closed `platform-gitops`#38 and `platform-gitops`#54).

## 2026-06-05 — Crossplane sync unblock — XRD admission + kubeconform Composition skip (rl)

- **Tool:** Claude Code (local, WSL/Debian, Opus 4.7)
- **Scope:** `INENI-PT-GROUP-B/platform-gitops`#57 — `crossplane/xrds/xtenant.yaml`
  and `.github/workflows/validate.yml`.
- **What:** Two bugs surfaced together after pp's `platform-gitops`#50 went live.
  (1) The XRD generated from `platform-gitops`#46 was rejected at admission because
  `additionalProperties: false` (which I had requested during the #46 review)
  clashes with the system-owned `spec` fields Crossplane injects at
  CRD-generation. Dropped it from both `spec` and `imageTagOverride`. A Marco PR
  review reframed my inline rationale away from a generic K8s rule to the
  Crossplane-injection cause; the comments were rewritten on a third commit
  before merge. (2) The datreeio CRDs-catalog at the pinned SHA ships the
  Crossplane v2 `Composition` schema; our install is v1.x P&T, so `kubeconform`
  rejected `platform-gitops`#55's `resources[]`. Added `Composition` to the
  `-skip` arg list with an inline rationale comment, same pattern as the existing
  `ProviderConfig` and `Object` skips.
- **Verification:** Catalog schema content checked via
  `curl ... | python3 -c "json.load(sys.stdin)..."`, confirming the v2 field set
  verbatim. `yamllint` exit 0 locally on both files.
- **Outcome:** merged — `platform-gitops`#57 merged 17:49 UTC, auto-closed
  `platform-gitops`#56. `platform-gitops`#55 was rebased + force-pushed to pick
  up the new `-skip` list and CI went green.

## 2026-06-05 — Recurring AI-assisted issue and PR review drafting (mr)

- **Tool:** Claude Code (local, WSL/Debian, Opus 4.7)
- **Scope:** cross-repo — `[TASK]` issue bodies and PR review comments
  drafted by `@marco93r` across `platform`, `platform-iac`,
  `platform-gitops`, `app-backend`, and `app-frontend`. Recurring
  practice, not tied to a single PR; logged here as a single entry
  covering the workflow rather than per occurrence.
- **What:** Claude is used as a drafting assistant for two recurring
  authoring tasks. (1) Issue bodies — given a short prompt describing
  the task, Claude drafts the four required sections (`Context`,
  `Scope`, `Acceptance Criteria`, `Grading Pillar`) in the org-wide
  `[TASK]` template shape from `INENI-PT-GROUP-B/.github`, with the
  GFM line-break and label conventions from `CONTRIBUTING.md`
  § Creating Issues already applied. (2) PR reviews — Claude reads the
  diff and the linked issue, then drafts review comments (correctness
  spot-checks, convention deviations, missing acceptance-criteria
  coverage). In both cases the AI output is a **draft, never a final
  artefact**.
- **Verification:** every draft is reviewed and adjusted before it
  leaves my machine — no draft is posted as-is. Issue bodies are
  checked against the `task.yml` template structure and the Grading
  Pillar enum, then edited for wording, scope tightening, and
  acceptance-criteria realism before `gh issue create`. Review-comment
  drafts are checked against the actual diff (no fabricated line
  references, no invented file paths) and against the project
  conventions in `docs/claude/`; comments that read plausibly but
  miss the real concern are rewritten or dropped, and the final
  approve / request-changes decision and prioritisation of comments
  are mine. Drafts that drift from the architecture-decisions record
  are corrected manually rather than accepted.
- **Outcome:** ongoing — recurring workflow; individual issues and PRs
  authored this way are not linked here to avoid bloating the log.

## 2026-06-03 — Root App spec-identity CI check (#29)

- **Tool:** Claude Code (local, Windows/PowerShell, Opus 4.7)
- **Scope:** `INENI-PT-GROUP-B/.github` — new
  `.github/workflows/root-app-spec-identity-reusable.yml` (PR
  `.github`#12). Two follow-up caller PRs in `platform-iac` and
  `platform-gitops` will wire the reusable from each side and close
  `platform-gitops`#29.
- **What:** Drafted the full `workflow_call` reusable that diffs the
  `spec` blocks of `platform-iac/bootstrap/argocd-bootstrap.yaml` and
  `platform-gitops/applications/root.yaml` on every PR that touches
  either file. Comparison normalises through
  `yq -o=json '.spec' | jq -S .` to canonical sorted-key JSON, then
  `diff -u`. `metadata.finalizers` diverges intentionally on the gitops
  side, so only `spec` is compared; a null-spec guard prevents false
  passes if the top-level key disappears. Cross-repo checkout uses raw
  `git clone` over HTTPS — both target repos are public, so no token
  plumbing. Defensive shell hygiene: every user-controlled value (the
  `pr-side` input and step outputs derived from it) is threaded through
  env vars instead of direct `${{ }}` interpolation into shell.
- **Verification:** Workflow shape and styling reviewed against the
  existing reusables in the same directory
  (`commitlint-reusable.yml`, `lint-reusable.yml`,
  `pr-title-reusable.yml`). yq pinned to v4.53.2 to match
  `platform-gitops/.github/workflows/validate.yml`. The two contract
  files were manually inspected to confirm their `spec` blocks are
  currently identical on `main` (i.e. the gate starts green). A
  deliberate-divergence negative test will run after this PR merges and
  the caller workflows go live in `platform-iac` / `platform-gitops`;
  the failing CI run URL will be linked in `platform-gitops`#29's
  closing comment.
- **Outcome:** open — `INENI-PT-GROUP-B/.github`#12, awaiting review.

## 2026-06-03 — provider-gcp install + Option A ADR (S2-09, #21)

- **Tool:** Claude Code (local, Windows/PowerShell, Opus 4.7)
- **Scope:** `platform-gitops` — new
  `crossplane/provider-installs/provider-family-gcp.yaml` and
  `crossplane/provider-installs/provider-gcp-secretmanager.yaml`;
  comment edits to
  `crossplane/providers/deployment-runtime-config-gcp.yaml` and
  `applications/crossplane-providers.yaml` (PR `platform-gitops`#48,
  closes `platform-gitops`#21, transitively retires
  `platform-gitops`#17). `platform` — new "Crossplane providers —
  family layout for GCP" subsection in
  `docs/claude/architecture-decisions.md` (companion PR `platform`#88).
- **What:** Drafted the two Provider CR files matching the existing
  project pattern (cf. `provider-helm.yaml`,
  `provider-kubernetes.yaml`), both pinned to v2.5.4. Drafted the DRC
  inline-comment update articulating the Option A trade-off (DRC name
  stays generic `provider-gcp` even though the controller pod ships in
  the sub-provider, per the per-Provider ownership constraint from
  crossplane/crossplane#4552), and drafted the architecture-decisions
  ADR subsection that records the decision. The rbac-manager reasoning
  for skipping a manual ClusterRoleBinding ("provider-gcp-secretmanager
  only manages its own MR CRDs in-cluster, unlike provider-helm /
  provider-kubernetes which need cluster-admin") was Claude-suggested
  and verified against `values/crossplane.yaml`
  (`rbacManager.replicas: 1`).
- **Verification:** Both package versions (v2.5.4 of
  `provider-family-gcp` and `provider-gcp-secretmanager`) verified
  live against the Upbound marketplace via `WebFetch` before pinning.
  The DRC-naming Option A decision itself was made by am with rl in
  `platform-gitops`#21's comment thread (rl deferred the final call to
  am 2026-05-31); the file-level write-ups (inline DRC comment, ADR
  subsection) are Claude-drafted in language that records the decision
  for the next reader. `kubeconform`, `yamllint`, `helm lint`, and
  `markdownlint` CI all green on both PRs.
- **Outcome:** open — `platform-gitops`#48 and `platform`#88,
  awaiting review.

## 2026-06-01 — Day-1 end-to-end validation (S2-08)

- **Tool:** Claude Code (local, WSL/Debian, Opus 4.7)
- **Scope:** `platform-iac` — new top-level `VALIDATION.md` + new `validation-day1/` directory with five files (`00-namespace.yaml`, `01-hello-app.yaml`, `02-hello-ingress.yaml`, `03-external-secret.yaml`, `README.md`). Pending PR `platform-iac`#53 (closes `platform-gitops`#23).
- **What:** Drafted the test fixtures (nginx Deployment + Service + ConfigMap, Ingress on `hello.fhuebung.lol`, ExternalSecret pulling a seeded GSM payload via ESO) and the `VALIDATION.md` proof document capturing AC1–AC4 evidence. The full validation procedure was executed twice on the live cluster — once by Claude Code via the local Bash tool (apply, capture outputs, cleanup), and once independently by the operator from a separate WSL shell using the documented commands verbatim as a reproducibility check.
- **Verification:** The operator's separate WSL re-run surfaced a hallucination episode in the first `VALIDATION.md` draft: parts of the captured-evidence sections that Claude had presented as transcribed from the test run did not match the real session logs — some lines were generated rather than copied from output. When asked to audit, Claude reviewed its own session transcript, acknowledged the inaccuracies, and could itself confirm from the logs which lines were truly captured and which had been invented. The doc was rewritten line by line against the verifiable session output, the operator re-ran the corrected procedure end-to-end, and the second run produced structurally identical AC1–AC4 outcomes (time-of-run fields aside). The episode is logged here because it is the failure mode the assignment expects us to surface: AI-generated content presented as verifiable evidence, detected only by an external check the operator ran outside the AI's own session.
- **Outcome:** open — `platform-iac`#53, awaiting review.

## 2026-06-01 — Day-1 and Day-2 architecture diagrams (S2-13)

- **Tool:** Claude Code (local, macOS, Opus 4.7)
- **Scope:** `platform` — new `docs/architecture/day1-bootstrap.drawio` +
  `.drawio.png`, new `docs/architecture/day2-tenant-provisioning.drawio` +
  `.drawio.png`, updates to `docs/architecture/README.md` describing both
  diagrams (#83, S2-13).
- **What:** Claude was used as a faster drafting tool for the initial
  `.drawio` XML scaffolding of both diagrams — swim-lane geometry, box
  placement, label markup, and arrow routing — instead of authoring the
  XML by hand. The substantive work was manual: deciding the phases
  themselves (which step belongs in which lane, where the cut between
  Day-1 and Day-2 sits, which sub-steps live inside the Phase 4
  container), choosing the wording so the diagrams stay self-explanatory
  without jargon, and coordinating the actors across the lanes so the
  arrows tell a coherent story. The Day-2 diagram was deliberately
  redesigned in the Day-1 swim-lane style after the first UML-sequence
  draft was set aside on readability grounds. Layout fine-tuning (title
  wording, separator characters, lane labels, lane widths) was done
  directly in draw.io Desktop. The README narrative for the diagrams
  was drafted with Claude's help and trimmed manually.
- **Verification:** XML parsed cleanly for both files before each PNG
  export. Day-1 was cross-checked against the actual `platform-gitops`
  state on `main` (eleven Argo CD-reconciled Applications match, only
  `kube-prometheus-stack` dashed) and against `platform-iac/terraform/dns/`
  for the `dns.reader` binding from `platform-iac`#44. Day-2 was
  cross-checked against `architecture-decisions.md` and the existing
  Tenant onboarding narrative. Three independent review passes from
  `@marco93r` (Day-1 component spot-check, `dns.reader`, no per-tenant
  cert-manager step), `@mlexinho27` (Day-2 as-merged vs documented
  design target, Traefik `Middleware` bridge in Phase 4, GHCR pull secret
  visibility, Phase 5a/5b README/diagram alignment), and `@ronaldley`
  (phase-for-phase against `bootstrap.sh`, IAM GSAs verbatim against
  `terraform/iam/main.tf`, all five Composition items per
  `architecture-decisions.md` § Multi-tenancy visible in Phase 4) were
  incorporated through hand-edited diagram and README adjustments, not
  by regenerating the files.
- **Outcome:** merged — `platform`#83 (closes #72).

## 2026-05-30 — External Secrets Operator (S2-05) and Argo CD Ingress wildcard rename

- **Tool:** Claude Code (local, macOS, Opus 4.7)
- **Scope:** `platform-gitops` — new `applications/external-secrets.yaml`,
  `values/external-secrets.yaml`,
  `applications/external-secrets-clustersecretstore.yaml`, and
  `external-secrets/clustersecretstores/gcp-secret-manager.yaml`;
  `.github/workflows/validate.yml` extension (#32, S2-05). `platform-iac`
  — `bootstrap/argocd-values.yaml` and `README.md` wildcard secret name
  alignment (#42, S2-01 follow-up).
- **What:** drafted ESO as a Day-1 platform component — a multi-source Argo
  CD Application installing the upstream `external-secrets` Helm chart
  (pinned 2.5.0), a minimal values file (Workload Identity annotation on
  the chart's ServiceAccount, requests + memory-only limits per component
  matching the other platform values files), and a sibling Application
  shipping a `ClusterSecretStore` (`external-secrets.io/v1`) that
  authenticates to Google Secret Manager via `auth.workloadIdentity`
  referencing the ESO KSA; the GSA (`external-secrets`, project-level
  `roles/secretmanager.secretAccessor`, provisioned in platform-iac S1-08)
  is wired through the chart's `serviceAccount` annotation. `validate.yml`
  was extended to cover the new `external-secrets/` directory in both the
  kubeconform path list and the trigger `paths:` filter. Separately, the
  wildcard TLS secret name in the Argo CD Ingress and its README
  description was aligned from `wildcard-fhuebung-lol` to
  `wildcard-fhuebung-lol-tls` to match the actual cert Secret produced by
  `platform-gitops/traefik/certificate.yaml`.
- **Verification:** chart version, `ClusterSecretStore` apiVersion (`v1`),
  the `gcpsm.auth.workloadIdentity` field shape, and the kubeconform
  catalog schema availability were verified directly against the chart and
  the pinned catalog SHA before writing. Local verification mirrored the
  CI gate — `helm lint --strict` clean, `kubeconform` valid with 0 errors,
  and `helm template` was used to confirm resources land on the
  controller, webhook and certController deployments, that the WI
  annotation sits on the ESO controller's ServiceAccount, and that the
  chart default `rbac.serviceAccountTokenCreate: true` makes the
  `serviceAccountRef` auth path runtime-viable. Review feedback from
  `@mlexinho27` was incorporated — `installCRDs: true` was pinned
  explicitly so a future chart-default flip cannot silently render the
  CRDs out of the release (same forward-safety reasoning as the chart
  version pin, consistent with the explicit `crds.enabled`/`crds.keep`
  pin in `values/cert-manager.yaml`). The wildcard-rename surfaced while
  reviewing the chart-contract doc (`platform-gitops`#33); the doc default
  was aligned with the real Secret name there, but the chart itself
  (`app-backend/chart/values.yaml`) still carries the un-aligned
  `wildcard-fhuebung-lol` and is tracked for a follow-up PR pair.
  End-to-end run against the cluster is deferred to S2-08.
- **Outcome:** merged — `platform-gitops`#32 (closes #13), `platform-iac`#42
  (closes #41).

## 2026-05-29 — Sprint 2 Argo CD work (S2-14 CI gate, S2-06 Traefik)

- **Tool:** Claude Code (local, WSL, Opus 4.7)
- **Scope:** `platform-gitops` — `.github/workflows/validate.yml`
  (#27, S2-14); `applications/traefik.yaml`,
  `applications/traefik-certificate.yaml`, `values/traefik.yaml`,
  `traefik/certificate.yaml` (#30, S2-06)
- **What:** drafted the `helm lint --strict` + `kubeconform` validation gate
  for the Argo CD manifests (discovers each Helm-sourced Application, pulls
  the pinned chart, lints against its `values/*.yaml`; schema-validates
  `applications/`, `cert-manager/`, `crossplane/providers/`, `tenants/`
  against the Kubernetes API and a pinned `datreeio/CRDs-catalog` commit;
  Crossplane XRDs/Compositions deferred to S3). Then the Traefik
  Application set: chart-install Application (Traefik 40.2.0), sibling
  Application for the CRD-dependent wildcard `Certificate` (decoupled with
  `SkipDryRunOnMissingResource=true`, mirroring `cert-manager-clusterissuers`),
  the `Certificate` itself, and Helm values (LoadBalancer with
  ExternalDNS hostname annotation, `web` → `websecure` redirect, default
  TLSStore referencing the wildcard Secret, Traefik's own ACME off).
- **Verification:** for #27, two review rounds were incorporated — the
  discovery loop was inverted from `values/*.yaml` to `applications/*.yaml`
  so the Application stays the source of truth for the chart-↔-values link,
  and `cert-manager/` was added to the trigger paths after a near-miss. For
  #30, the chart 40.2.0 value keys were verified against `helm pull`,
  `helm lint --strict` reported `0 chart(s) failed`, and the cross-file
  references (`letsencrypt-prod` issuer, Certificate Secret ↔ TLSStore,
  ExternalDNS annotation contract) were checked locally. Live verification
  (LB IP, DNS record, `Ready=True`) is gated on S2-08.
- **Outcome:** merged — `platform-gitops`#27 (closes #26). Open —
  `platform-gitops`#30 (closes #25), pending review.

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
