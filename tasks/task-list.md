# Task List — INENI-PT-GROUP-B

> **Generated:** 2026-05-20  
> **Last revised:** 2026-05-21  
> **Deadline:** 2026-06-26, 14:00 CEST  
> **Team:** mr (Marco), am (Alex), rl (Ronny), pp (Patrick)  
> **Domain:** `fhuebung.lol` · **GCP Project:** `dotted-axle-495612-f4`

---

## Status of already completed work

| Done | Where |
|---|---|
| Architecture Design and Decissions | GitHub |
| Budget and Cost Planning | GCP Calculator |
| GitHub Org + all 6 repos created | GitHub |
| Branch protection, commitlint, pr-title CI | platform-iac, platform-gitops, platform, app-backend, app-frontend (Branch protection on convention -> FREE Github plan) |
| `providers.tf`, `variables.tf`, `terraform.tfvars.example` | platform-iac/terraform |
| platform-gitops directory structure (`.gitkeep`) | platform-gitops |
| CONTRIBUTING.md | platform |

> **Note on app-backend / app-frontend:**   The actual app implementation, chart authoring, and first release tag are tracked in #32. Tasks in this list that reference these repos (S1-03, S1-04, S1-11, S1-12, S2-12) define *platform-side management* (CI scaffolding, image/chart publish pipelines, registry strategy). 

---

## Sprint 1 — Platform Services & CI Foundations (22.–28.05.)

> **Goal:** Cluster exists, ArgoCD is reachable, all linting CI green, Terraform state
> backend in place, Workload Identity wired for in-cluster components. This sprint
> unblocks everything downstream.

### Step 1 — Linting CI in all repos (must land BEFORE any code module is opened in PR)

> Rationale: lint pipelines are the cheapest hygiene win and must guard every subsequent
> code-bearing PR. They have no dependencies, are small workflow files, and unblock
> nothing — so we land them first across all repos, in parallel.

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S1-01 | Add GitHub Actions workflow in **platform-iac**: on push/PR touching `**/*.tf` run `terraform fmt -check` + `terraform validate` + `tflint`. **No `plan`, no `apply` in CI** — all deployments happen exclusively via `bootstrap.sh` run by a team member locally. Hence no OIDC / WIF binding for Terraform CI is needed. | **rl** | P1+P2 | — |
| S1-02 | Add `yamllint` + `markdownlint` CI workflow to **platform-gitops** | **pp** | P1 | — |
| S1-03 | Add `yamllint` + `markdownlint` CI workflow to **app-backend** | **am** | P1 | — |
| S1-04 | Add `yamllint` + `markdownlint` CI workflow to **app-frontend** | **am** | P1 | — |

### Step 2 — Terraform state backend (chicken-and-egg solved via bootstrap script) + network module

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S1-05 | Write `bootstrap/phase_2_state.sh` (or inline phase in `bootstrap.sh`) — `gcloud storage buckets create gs://${PROJECT}-tfstate --location=europe-west1 --uniform-bucket-level-access`, enable versioning, idempotent (skip if bucket exists). Then add `backend.tf` referencing that bucket. Document the order in `platform-iac/README.md`: bootstrap script creates the bucket first, then `terraform init` succeeds. | **mr** | P2 | S1-01 |
| S1-06 | Add Terraform module `terraform/network/` — VPC, subnet, firewall rules | **rl** | P2 | S1-01, S1-05 |

### Step 3 — GKE cluster + Workload Identity

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S1-07 | Add Terraform module `terraform/cluster/` — GKE Standard, zonal, WI enabled, Dataplane V2, autoscaling 3–6 × e2-standard-4 | **mr** | P2 | S1-06 |
| S1-08 | Add Terraform module `terraform/iam/` — Workload Identity pool, SA bindings for in-cluster workloads (ExternalDNS, cert-manager, ESO, Crossplane). No GitHub OIDC federation needed (Terraform runs only locally via `bootstrap.sh`). | **pp** | P2 | S1-07 |
| S1-08a | Add Terraform resource for GCS backup bucket `gs://${PROJECT}-pg-backups` (location europe-west1, uniform bucket-level access, versioning off, lifecycle rule: delete objects > 30 days). IAM binding so tenant-namespace ServiceAccounts can write under their own `<tenant>/` prefix via WI. | **rl** | P2 | S1-08 |

### Step 4 — DNS zone re-created via Terraform

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S1-09 | Manually delete the existing Cloud DNS zone `platform-zone` (created out-of-band), then create it fresh via Terraform with identical config (name `platform-zone`, DNS name `fhuebung.lol.`, public visibility, DNSSEC off). Reason: importing would break the one-click `bootstrap.sh` deployment promise. After re-creation, check the new NS records and update them at Porkbun if they differ. Add IAM binding `roles/dns.admin` (scoped to the zone) for ExternalDNS + cert-manager SAs. Update `DNS_SETUP.md` to reflect that the zone is now Terraform-managed. | **am** | P2 | S1-08 |

### Step 5 — Bootstrap script (assembles everything above into a one-command deployment)

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S1-10 | Write `bootstrap/bootstrap.sh` — phases 0–5 (preflight, API enable, **integrates the state-bucket creation from S1-05**, terraform apply, kubeconfig, argocd bootstrap), idempotent, logs to `bootstrap.log`, reads from `bootstrap.env` (gitignored, `bootstrap.env.example` committed) | **mr** | P2 | S1-05, S1-09 |

### Step 6 — App CI pipelines (parallel to terraform work)

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S1-11 | Add GitHub Actions workflow in app-backend: lint + test + Docker build + GHCR push (public image `ghcr.io/ineni-pt-group-b/app-backend`); tags: `latest`, `sha-<short>`, `v*`. Pipeline scaffolding; concrete Node.js/Fastify build steps land via #32. | **am** | P1+P3 | S1-03, #32 Phase 1 |
| S1-12 | Add GitHub Actions workflow in app-frontend: build + Docker build + GHCR push (private image `ghcr.io/ineni-pt-group-b/app-frontend`); same tag strategy. Pipeline scaffolding; concrete Vite/React build steps land via #32. | **pp** | P1+P3 | S1-04, #32 Phase 1 |

### Step 7 — Hygiene docs

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S1-13 | Check and add AI_USAGE entries (log architecture session, bootstrap script generation, docs drafts) | **rl** | P1 | — |
| S1-14 | Add `CODEOWNERS` to all 6 repos (platform, .github, platform-iac, platform-gitops, app-backend, app-frontend) | **mr** | P1 | — |
| S1-15 | Write `platform/docs/cost-planning/` — initial capacity estimate (node sizing, cost per month, dated snapshot) | **pp** | P1 | — |



---

## Sprint 2 — ArgoCD, Platform Services & Crossplane Install (29.05.–04.06.)

> **Goal:** Cluster is fully bootstrapped via GitOps. cert-manager issues a valid wildcard
> cert, ExternalDNS writes records, Traefik serves HTTPS. Crossplane and the app Helm chart
> are ready to go.

### Step 8 — ArgoCD bootstrap (depends on S1-10 cluster running)

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S2-01 | Write `bootstrap/argocd-bootstrap.yaml` — installs ArgoCD via Helm (incl. Helm values for the `argocd.fhuebung.lol` Ingress with wildcard TLS) + root App-of-Apps pointing at `platform-gitops/applications/` | **pp** | P2 | S1-10 |
| S2-02 | Write `applications/root.yaml` — root Argo CD Application (self-managing, prune+selfHeal) | **am** | P2 | S2-01 |

### Step 9 — Platform component ArgoCD Applications (depends on S2-02)

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S2-03 | Write ArgoCD Application + Helm values for **cert-manager** (ClusterIssuer: staging + prod, DNS-01 via Cloud DNS, WI-bound SA) | **am** | P2 | S2-02, S1-09 |
| S2-04 | Write ArgoCD Application + Helm values for **ExternalDNS** (Cloud DNS provider, WI, `txtOwnerId=gke-prod` per architecture diagram) | **rl** | P2 | S2-02, S1-09 |
| S2-05 | Write ArgoCD Application + Helm values for **External Secrets Operator** + ClusterSecretStore (Google Secret Manager backend, WI) | **pp** | P2 | S2-02, S1-08 |
| S2-06 | Write ArgoCD Application + Helm values for **Traefik** (LoadBalancer Service, wildcard cert `*.fhuebung.lol`). IngressRoutes for `argocd.fhuebung.lol` and `grafana.fhuebung.lol` are part of the respective component Helm values (S2-01 area for Argo CD, S4-01 for Grafana), not this task. | **mr** | P2 | S2-02, S2-03 |
| S2-07 | Write ArgoCD Application + Helm values for **CloudNativePG operator** (namespace `cnpg-system`) | **am** | P2 | S2-02 |

### Step 10 — End-to-end Day-1 validation

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S2-08 | Validate: `hello.fhuebung.lol` returns 200 with valid Let's Encrypt cert; DNS record written by ExternalDNS; secret round-trip from Google Secret Manager to Kubernetes Secret works | **rl** | P2 | S2-03, S2-04, S2-05, S2-06 |

### Step 11 — Crossplane install + app chart contract

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S2-09 | Write ArgoCD Application for **Crossplane core** + provider-gcp + provider-helm + provider-kubernetes (Helm install). provider-gcp is needed for writing per-tenant BasicAuth htpasswd entries into GSM. | **am** | P3 | S2-02 |
| S2-10 | Write Crossplane ProviderConfigs (`crossplane/providers/`) — ProviderConfig for provider-gcp (WI-bound SA with `roles/secretmanager.secretVersionAdder` on per-tenant prefix), provider-helm and provider-kubernetes (in-cluster ServiceAccount) | **rl** | P3 | S2-09 |
| S2-11 | Document the chart contract between #32 and the Crossplane Composition: pin the `values.yaml` surface (`tenant`, `host`, `image.backend.{repository,tag}`, `image.frontend.{repository,tag}`, `ingress.{className,tls.secretName,basicAuthMiddlewareRef}`, `imagePullSecrets`) in `platform-gitops/docs/chart-contract.md`. Changes on either side require a coordinated PR. | **mr** | P3 | #32 Phase 4 |
| S2-12 | Add `helm push oci://ghcr.io/ineni-pt-group-b/app-chart` step to the app-backend release workflow. The chart sources land via #32 Phase 4; this task only wires the push step into the CI scaffolding from S1-11. | **rl** | P3 | S2-11, S1-11, #32 Phase 4 |

### Step 12 — Docs

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S2-13 | Write architecture diagrams — `platform/docs/architecture/` — Day-1 bootstrap phases flow diagram + Day-2 tenant provisioning sequence diagram. The existing logical_architecture diagram stays and may be updated if components changed. | **pp** | P1 | S2-08 |
| S2-14 | Add `kubeconform` + `helm lint` CI to **platform-gitops** | **mr** | P1 | S2-02 |

---

## Sprint 3 — Crossplane Compositions & Tenant Provisioning (05.–11.06.)

> **Goal:** A 10-line YAML claim in `tenants/` produces a fully running tenant under
> `<name>.fhuebung.lol` with its own namespace, database, secrets, and app — end to end.

### Step 13 — XRD definition (depends on S2-10 provider configs)

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S3-01 | Write Crossplane XRD `XTenant` (`crossplane/xrds/xtenant.yaml`) — fields: `name`, `tier` (small/medium), `version`, optional `imageTagOverride` | **mr** | P3 | S2-10 |

### Step 14 — Composition implementation (all depend on S3-01)

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S3-02 | Composition `xtenant-default` — Namespace + ResourceQuota + LimitRange per tenant | **pp** | P3 | S3-01 |
| S3-03 | Composition — NetworkPolicies (default-deny ingress/egress + explicit allow: tenant pods → DB, ingress → pods) | **rl** | P3 | S3-01 |
| S3-04 | Composition — CloudNativePG `Cluster` per tenant (1 replica, GKE SSD storage class, backup config pointing at `gs://${PROJECT}-pg-backups/<tenant>/`). | **am** | P3 | S3-01, S2-07, S1-06a |
| S3-05 | Composition — BasicAuth setup per tenant: (a) generate random password, bcrypt-hash it into htpasswd format (`admin:<bcrypt-hash>`), write to GSM at `tenant-<name>/basicauth-htpasswd` via provider-gcp; (b) `ExternalSecret` pulling htpasswd into a K8s Secret with the `users` key (Traefik format); (c) `ExternalSecret` pulling the shared GHCR pull secret from GSM (`shared/ghcr-pull-secret`); (d) Traefik `Middleware` (BasicAuth) referencing the htpasswd Secret. | **pp** | P3 | S3-01, S2-05, S2-10 |
| S3-06 | Composition — Helm `Release` via provider-helm (OCI chart from GHCR, image tags from `values/app-version.yaml`, `imageTagOverride` per claim, env vars: `DATABASE_URL`, `PORT` only — no app-level auth). Ingress carries `traefik.ingress.kubernetes.io/router.middlewares: tenant-<name>-basicauth@kubernetescrd` annotation wiring the Middleware from S3-05. | **mr** | P3 | S3-01, S3-05, S2-12 |

### Step 15 — Central app version file + staging tenant

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S3-07 | Write `values/app-version.yaml` — central default image tags + chart version; Composition reads this value when no `imageTagOverride` is set | **rl** | P3 | S3-06 |
| S3-08 | Write `tenants/staging.yaml` — first tenant claim with `imageTagOverride` set; validate end-to-end: `staging.fhuebung.lol` reachable with BasicAuth prompt and valid cert, own namespace and DB | **am** | P3 | S3-07, S3-02–S3-06 |

### Step 16 — Multi-tenancy validation

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S3-09 | Add tenants `acme` + `globex`; verify cross-tenant DB access is blocked, ResourceQuotas enforced, NetworkPolicies isolate traffic; document test output in repo (`docs/multi-tenancy-validation.md`) | **rl** | P3 | S3-08 |
| S3-10 | Verify global app-version rollout: update `values/app-version.yaml`, confirm all tenants without override roll to new version; document | **pp** | P3 | S3-08 |

---

## Sprint 4 — Monitoring, Polish & Lifecycle Tests (12.–18.06.)

> **Goal:** Monitoring is live and useful. Lifecycle scenarios are tested. Documentation is
> complete except for final actuals.

### Step 17 — Monitoring stack

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S4-01 | Write ArgoCD Application for **kube-prometheus-stack** (Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics; Prometheus PVC ~20 GiB, 15-day retention; Grafana Ingress for `grafana.fhuebung.lol` with wildcard TLS) | **am** | P3 (Bonus) | S2-02 |
| S4-02 | Write Grafana dashboards as ConfigMaps (`applications/monitoring/`) — Platform-Admin-View (cluster-wide: node CPU/mem, pod count, PVC usage) + App-Owner-View (namespace-scoped: backend latency, DB connections, pod restarts) | **mr** | P3 (Bonus) | S4-01 |

### Step 18 — Lifecycle tests

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S4-03 | Test + document tenant deletion flow (remove claim from `tenants/`, verify namespace + DB + DNS record cleaned up) | **am** | P3 | S3-09 |
| S4-04a | Test + document app crash recovery: kill backend pod, verify readiness probe triggers restart, tenant stays reachable. ~30 min. | **pp** | P3 | S3-09 |
| S4-04b | Test + document cluster redeploy: full teardown + rerun `bootstrap.sh`. Document timing, any manual steps, what persists (GSM, GCS) vs. what is rebuilt, post-rebuild image build/chart push triggers. Useful for the demo's "scaling outlook". | **pp** | P3 | S4-04a |

### Step 19 — Cost actuals + doc pass

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S4-05 | Update `platform/docs/cost-planning/` — add billing screenshot actuals, compare to initial estimate | **pp** | P1 | S4-04b |
| S4-06 | Write tenant onboarding guide (`platform-gitops/docs/tenant-onboarding.md`) — step-by-step: create claim file, open PR, what happens next, how to verify | **am** | P3 | S3-08 |
| S4-07 | Update all READMEs (platform, platform-iac, platform-gitops, app-backend, app-frontend) — reflect final state, quickstart commands, link to all docs | **rl** | P1 | S4-03 |
| S4-08 | Write `AI_USAGE.md` final entries (Compositions, Helm chart, troubleshooting sessions) | **mr** | P1 | — |

---

## Sprint 5 — Docs, Presentation & Submission (19.–26.06.)

> **Goal:** Final submission before 2026-06-26 14:00 CEST. Two dry runs done. Demo backup
> video recorded.

### Step 20 — Presentation prep

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S5-01 | Write presentation slides — architecture overview, tool decisions, grading pillar coverage (max. 12 slides, 20 min) | **mr** | P4 | S4-08 |
| S5-02 | Write demo script — live tenant creation: PR → ArgoCD sync → Crossplane reconcile → tenant reachable under `demo.fhuebung.lol` | **rl** | P4 | S3-08 |
| S5-03 | Write demo slides — app walkthrough, Grafana dashboards, multi-tenancy proof (NetworkPolicy block screenshot) | **am** | P4 | S4-01, S3-09 |
| S5-04 | Prepare capacity actuals + scaling forecast slides — billing screenshots, cost-per-tenant calculation | **pp** | P4 | S4-05 |

### Step 21 — Dry runs + backup

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S5-05 | Dry run 1 — full walkthrough internally (all 4 present), collect feedback, fix gaps | **all** | P4 | S5-01–04 |
| S5-06 | Dry run 2 + record demo backup video (worst-case contingency) | **all** | P4 | S5-05 |

### Step 22 — Lecturer cluster access (final pre-submission step)

> **Critical:** The assignment explicitly requires lecturer cluster-admin access.
> This is intentionally placed late so the cluster is in its final state, but
> **must not be skipped** before submission. Confirm the lecturer's GCP identity
> (e-mail) early via Moodle so the binding can be applied without delay.

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S5-06a | Grant `@muhlba91` cluster-admin: (a) confirm lecturer's GCP identity (email) via Moodle / e-mail; (b) add Terraform resources in `platform-iac/terraform/iam/`: project-level binding `roles/container.viewer` for GKE control-plane visibility, plus a `ClusterRoleBinding` to `cluster-admin` for that identity (applied via Argo CD as a Kubernetes manifest in `platform-gitops/applications/cluster-access/`); (c) verify by asking the lecturer to run `kubectl get nodes`; (d) document in main README. | **mr** | P2 | S5-06 |

### Step 23 — Final submission check

| # | Task | Assignee | Pillar | Depends on |
|---|---|---|---|---|
| S5-07 | Final repo audit: all repos public/private as required, no secrets in history, lecturer has GitHub admin access AND cluster-admin (verified in S5-06a), all CI green, all open PRs merged or closed | **mr** | P1 | S5-06a |

---

## Overall task count per person

| Person | P1 | P2 | P3 | P4 | Total |
|---|---|---|---|---|---|
| **mr** | 4 | 5 | 4 | 1+shared | **14** |
| **am** | 3 | 4 | 7 | 1+shared | **14** |
| **rl** | 3 | 5 | 5 | 1+shared | **13** |
| **pp** | 5 | 3 | 6 | 1+shared | **14** |

> Per-pillar counts include cross-pillar tasks (e.g. S1-11 contributes to both
> P1 and P3); the "Total" column counts unique tasks. Workload is balanced
> across the team after the latest reassignments. mr carries the bootstrap
> script and lecturer-access task as single large items in P2.

---

## Critical path

```
S1-01 Terraform lint CI (gate for all .tf PRs)
  └── S1-05 state bucket (gcloud) + backend.tf
        └── S1-06 VPC/network
              └── S1-07 GKE cluster
                    └── S1-08 Workload Identity + IAM
                          ├── S1-06a GCS backup bucket
                          └── S1-09 DNS zone (recreate via Terraform)
                                └── S1-10 bootstrap.sh
                                      └── S2-01 ArgoCD bootstrap
                                            └── S2-02 root App-of-Apps
                                                  ├── S2-03 cert-manager
                                                  ├── S2-04 ExternalDNS      → S2-08 Day-1 validation
                                                  ├── S2-05 ESO
                                                  ├── S2-06 Traefik
                                                  └── S2-09 Crossplane (core + 3 providers)
                                                        └── S2-10 ProviderConfigs
                                                              └── S3-01 XRD XTenant
                                                                    ├── S3-02 Composition: Namespace/Quota
                                                                    ├── S3-03 Composition: NetworkPolicies
                                                                    ├── S3-04 Composition: CloudNativePG  ← also depends on S1-06a
                                                                    ├── S3-05 Composition: BasicAuth (htpasswd → GSM → ESO → Middleware)
                                                                    └── S3-06 Composition: Helm Release (env: DATABASE_URL, PORT only)
                                                                          └── S3-07 app-version.yaml
                                                                                └── S3-08 staging tenant LIVE ✅
```

Everything after `S3-08` (multi-tenancy validation, monitoring, lifecycle tests,
docs, presentation, lecturer access) is parallelisable and not on the critical path.
