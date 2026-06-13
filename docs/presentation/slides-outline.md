# Presentation Outline — INENI-PT-GROUP-B

> Working draft for task **S5-01** (architecture slides) covering the full
> 20-minute presentation. Companion artefacts: the demo script (S5-02,
> Ronny), the app/Grafana demo slides (S5-03, Alex), the capacity actuals
> slides (S5-04, Patrick). This outline reserves space for those slides but
> does not author them.
>
> **Hard cap:** 20 minutes, max. 12 slides per the course brief.
> **Elective topic chosen:** *Application health and resources monitoring*
> (one of the three options the assignment lets us pick).

---

## Time budget

| Block                              | Owner | Minutes | Slides |
| ---------------------------------- | ----- | ------- | ------ |
| 1. Title + agenda                  | mr    | 0:30    | 1      |
| 2. Platform overview               | mr    | 2:00    | 1      |
| 3. Capacity & cost — plan vs real  | pp    | 2:30    | 1–2    |
| 4. Live demo: new tenant           | rl    | 5:30    | 1–2    |
| 5. Scaling forecast                | mr    | 2:00    | 1      |
| 6. Elective: monitoring page demo  | am    | 4:30    | 2      |
| 7. Wrap-up + Q&A pointer           | mr    | 1:00    | 1      |
| **Total**                          |       | **18:00** | **10–12** |

Two-minute buffer for transitions and the inevitable demo hiccup. The deck
sits at the upper bound (12) only if the cost block and the demo each need
a second slide; default is 10.

---

## Slide-by-slide outline

### Slide 1 — Title (mr, 0:30)

- Project: *Multi-tenant SaaS platform on GKE*
- Team: Marco, Alex, Ronny, Patrick
- Course: Infrastructure Engineering, HS Burgenland SS 2026
- One-liner: "On-demand, isolated SaaS instances provisioned via GitOps and
  Crossplane."

### Slide 2 — Platform overview (mr, 2:00)

Visual: `platform/docs/architecture/logical_architecture.drawio.png` as the
single background image.

Talking points (do not put on the slide, deliver verbally):

- Six repositories, public except `app-frontend` (assignment requirement).
- One bootstrap command (`bootstrap.sh`) → Day 1 cluster, then Argo CD owns
  Day 2 from `platform-gitops`. No further manual clicks.
- Tool stack we landed on, in the order it shows up at runtime:
  Terraform → GKE Standard zonal → Argo CD → Crossplane (1.x, P&T
  Compositions) → CloudNativePG, ESO, ExternalDNS, cert-manager, Traefik,
  kube-prometheus-stack.
- *Why these choices in one sentence each* — defer detail to the docs; the
  slide just lists them.

On-slide labels only: tool names overlaid on the diagram regions they own
(IaC, GitOps control plane, in-cluster operators, tenant namespace).

### Slide 3 — Capacity & cost: plan vs. real (pp, 2:30)

**Owned by Patrick (S5-04).** This outline only reserves the slot and lists
the data sources Patrick should pull from so the numbers line up with the
rest of the deck.

Plan side (from `platform/docs/cost-planning/capacity_costs_gcp.md`):

| Category                           | EUR/month |
| ---------------------------------- | --------- |
| Compute (GKE + Persistent Disk)    | 392.27    |
| Networking (LB + NAT)              | 23.41     |
| DNS + Secrets                      | 3.35      |
| Operations (Logging + Monitoring)  | 22.23     |
| Storage (TF state)                 | 0.02      |
| **Total estimate**                 | **441.27** |
| Granted budget                     | 500.00    |

Real side: GCP billing screenshot for the active window, broken down by the
same SKU groups. Patrick to fill in once billing data for the demo window
is final.

Cost-per-tenant: total / # tenants live during the active window. Call out
the dominant variable cost (current expectation: per-tenant CNPG Pods and
PVCs; the platform baseline is fixed).

### Slide 4 — Live demo: create a new tenant (rl, 5:30)

**Owned by Ronny (S5-02).** This outline reserves the slot, names the path
the demo follows, and lists the code surfaces to show. The actual demo
script is Ronny's deliverable.

Path:

1. Open PR on `platform-gitops` adding `tenants/demoLive.yaml` (an
    `XTenant` claim — `tier: small`).
2. Squash-merge. Argo CD picks up the new manifest on its next sync.
3. Crossplane reconciles the claim against the Composition
    `crossplane/compositions/xtenant-default.yaml` (~1340 lines, P&T) which
    creates, in one step:
    - Namespace + NetworkPolicy + ResourceQuota + LimitRange
    - CloudNativePG `Cluster` (with WI-federated backup IAM binding on
      `gs://<project>-pg-backups/<tenant>/`)
    - GHCR pull `ExternalSecret`
    - BasicAuth: random password → GSM `SecretVersion` → `ExternalSecret`
      → Traefik `Middleware` → ingress annotation
    - Helm `Release` via `provider-helm`, chart pulled from
      `ghcr.io/ineni-pt-group-b/app-chart`
4. ExternalDNS publishes the record; cert-manager has the wildcard
    already; the tenant URL `demoLive.fhuebung.lol` responds (BasicAuth
    prompt → app reachable).

Code to show on screen (open in IDE, not as slide images):

- `platform-gitops/tenants/demotenant1.yaml` — to show the claim shape
  before merging the new one.
- A folded view of `crossplane/compositions/xtenant-default.yaml` showing
  the resource list, then expand one or two of the more interesting patches
  (BasicAuth, Helm Release).
- Argo CD UI: the new `XTenant` going from `OutOfSync` → `Synced`.
- Crossplane: `kubectl get xtenant,managed -A` resolving.

Fallback: pre-recorded demo backup video (S5-06) if anything blocks live.

### Slide 5 — Scaling forecast: hundreds to thousands of tenants (mr, 2:00)

One slide, three columns: current / 100 tenants / 1000 tenants.

Per column list what changes:

**Today (3–6 tenants, baseline):**

- 3 × e2-standard-4 worker nodes (12 vCPU, 48 GB).
- 1 CNPG instance + 1 backend Pod + 1 frontend Pod per tenant.
- Single zonal GKE control plane.
- Single Crossplane Composition (`small` / `medium` tier).

**100 tenants — incremental:**

- Cluster Autoscaler grows the existing node pool (already configured 3–6;
  raise upper bound to ~20). Same instance type.
- Same Composition; no architectural change. Tier `medium` carries the
  bigger workloads.
- CNPG: still one cluster-per-tenant, but the per-tenant Pod count starts to
  dominate the kubelet density (~100 namespaces × ~5 Pods). Stays under
  GKE node limits.
- Logging cost scales linearly with tenant count — biggest variable cost
  item to watch.

**1000 tenants — architectural shifts required:**

- Move to **regional GKE** for control-plane SLA, not zonal.
- Multiple node pools by workload class: database-pool (high IOPS PDs) vs
  app-pool (compute).
- CNPG: keep one cluster per tenant up to a per-node Pod-density ceiling,
  then either shard tenants across multiple GKE clusters (one platform →
  multiple cells) or switch the smallest tier to a *shared* CNPG cluster
  with database-per-tenant.
- Argo CD: enable `ApplicationSet` sharding or split tenants across Argo CD
  instances; one Argo CD is fine to ~thousands of Applications but the
  sync-controller becomes the bottleneck.
- Secret Manager: per-tenant secret count is bounded (BasicAuth + DB creds
  + pull secret = ~3); 1000 tenants × 3 = 3000 versions — still trivial.
- Egress and load-balancer pricing become the next variable cost cliff.

Explicit assumption on the slide: linear extrapolation assumes tenants stay
on the `small` tier and traffic patterns match the demo workload.

### Slide 6 — Elective topic: Monitoring (am, 2:30 of 4:30)

**Owned by Alex (S5-03 covers the Grafana screenshots).** Slide content:

- Stack: `kube-prometheus-stack` Helm chart, self-hosted in-cluster.
- Exposed at `grafana.fhuebung.lol` via Traefik + wildcard cert.
- Workload Identity for any GCP-side reads; no SA-JSON anywhere.
- Dashboards loaded via Argo CD from
  `platform-gitops/applications/kube-prometheus-stack-dashboards.yaml`.

What we monitor:

- **Cluster health:** node CPU/memory/disk, kubelet pressure, autoscaler
  events.
- **Platform components:** Argo CD sync status, Crossplane reconcile loop
  durations, ESO secret-sync errors, cert-manager certificate expiry.
- **Tenant workloads:** per-namespace CPU/memory vs `ResourceQuota`,
  CNPG cluster health (replication lag, WAL archive backlog), HTTP error
  rates from Traefik per tenant ingress.

### Slide 7 — Elective topic: Monitoring (am, 2:00 of 4:30) — live page

Switch to browser, walk the audience through:

1. Grafana → "Tenant overview" dashboard, filter to one tenant.
2. Same dashboard during an artificial load spike (Ronny triggers a
    small load test from the demo tenant during the talk, prepared
    beforehand).
3. Argo CD UI showing all `Application`s green; click into
    `kube-prometheus-stack` to show the chain.
4. One alert example (e.g. `KubePodCrashLooping`) — silenced before the
    demo, shown as a routing example.

Code to point at briefly: the dashboard ConfigMap loader and the
`PrometheusRule` for tenant-level alerts.

### Slide 8 — Wrap-up + Q&A (mr, 1:00)

- The four grading pillars and where we land:
  - **Documentation & Hygiene:** Conventional Commits, squash-only, public
    repos, `AI_USAGE.md`, CONTRIBUTING.md.
  - **Infrastructure Bootstrap:** one-command from zero to running cluster.
  - **Application Management:** PR-driven tenant onboarding, central
    version-bump file `values/app-version.yaml` for one-shot fleet
    rollouts, `staging` tenant for testing first.
  - **Presentation:** this deck.
- Out-of-scope, on purpose: Argo Rollouts, frontend-to-GCS, Kyverno, hard
  multi-tenancy.
- Where to look: `INENI-PT-GROUP-B` GitHub org, lecturer has GitHub admin
  and cluster-admin (verified in S5-06a).
- Q&A pointer; we keep the cluster live for any post-talk probing.

---

## Coordination notes

- **Numbering authority:** this file. If Patrick (S5-04), Ronny (S5-02), or
  Alex (S5-03) move a slide, update the table in *Time budget* in the same
  PR.
- **Style:** match whatever template the deck already uses; do not
  introduce a second visual language.
- **Speaker handoff:** named owner per block; rehearse the transition
  sentence at dry run 1 (S5-05).
- **Diagrams:** reuse the existing `platform/docs/architecture/*.drawio.png`
  exports; do not rebuild them in slide-tool drawing layers.
- **Demo failure plan:** if live tenant create blocks, switch to the
  recorded backup video (S5-06).

## Open items before dry run 1 (S5-05)

- [ ] Patrick: fill in actuals for Slide 3 once billing window closes.
- [ ] Ronny: lock the live-demo tenant name and the path of the PR.
- [ ] Alex: confirm Grafana dashboards (S4-01) are import-ready and the
       "Tenant overview" dashboard exists.
- [ ] Marco: convert this outline into the actual slide deck and circulate
       a read-only link in the team chat.
- [ ] All: agree on a single fallback narrator if one of us is unavailable
       on the day.
