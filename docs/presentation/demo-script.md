# Tenant Onboarding Demo Runbook

> **Scope.** End-to-end demonstration of tenant onboarding on the platform for an
> audience seeing the system live for the first time (stakeholder, lecturer,
> new team member). Reusable across many contexts.
>
> **Out of scope.** Steady-state operator work — tier policy, name rules,
> `imageTagOverride` workflow, full verification matrix, credential handover,
> offboarding semantics. All of that lives in the canonical operator
> reference: [`platform-gitops/docs/tenant-onboarding.md`](https://github.com/INENI-PT-GROUP-B/platform-gitops/blob/main/docs/tenant-onboarding.md).
> This document is the **demo-specific overlay** — pre-flight checks,
> choreography, sync-acceleration, and fallback. It links out for everything
> else.
>
> **Also out of scope.** The presentation-slot-specific addendum for the final
> 20-minute course presentation (timing for the 5:30 min slot, slide cues,
> load-test handoff to Grafana) is intentionally not in this document and is
> tracked separately.

## When to use this runbook

- Ad-hoc demonstration to a stakeholder, lecturer, or new team member.
- Reproducible "show the system works" walkthrough before a code review or
  hand-over.
- Source material for the live demo block of the final presentation
  (with the presentation addendum on top).

Do **not** use this runbook to onboard a real tenant. Use
[`tenant-onboarding.md`](https://github.com/INENI-PT-GROUP-B/platform-gitops/blob/main/docs/tenant-onboarding.md)
for that.

## Tenant name convention

Throughout this runbook the tenant name `livedemoNN` is a **placeholder**.
Substitute `NN` with a fresh two-digit number for each run — e.g. `01` for
the first dry run, `02` for the second, `03` for the final presentation, and
so on.

A fresh name per run matters because the Composition's GSM secrets carry
`deletionPolicy: Orphan` — re-using a previous name inherits the prior
secret history. A fresh number gives you a clean tenant identity every time.

In every code block below, replace `livedemoNN` with the actual name
(e.g. `livedemo03`) before running the command.

## Pre-flight checklist

Run these checks **before the audience is in the room**. Each takes seconds.

1. **kubectl context** points at the platform GKE cluster:

    ```bash
    kubectl config current-context
    # → gke_dotted-axle-495612-f4_europe-west1-b_platform-cluster
    ```

    If not, refresh:

    ```bash
    gcloud container clusters get-credentials platform-cluster \
      --zone europe-west1-b \
      --project dotted-axle-495612-f4
    ```

2. **Argo CD UI** reachable and logged in. Open
    <https://argocd.fhuebung.lol> in a browser tab and confirm the
    Application list renders.

3. **No stale `livedemoNN` artefacts** in the cluster:

    ```bash
    kubectl get tenant -A 2>/dev/null | grep -q '^crossplane-system *livedemoNN ' \
      && echo "STALE — clean up first" || echo "clean"
    kubectl get namespace tenant-livedemoNN 2>/dev/null > /dev/null \
      && echo "STALE NS — clean up first" || echo "clean"
    ```

    If anything is stale: run the **Cleanup** section below for the previous
    `livedemoNN`, let it offboard, then re-run this check.

4. **Wildcard certificate** present and `Ready`. The cluster carries a single
    `*.fhuebung.lol` wildcard, so no per-tenant ACME wait is needed:

    ```bash
    kubectl get certificate -A
    # NAMESPACE   NAME                    READY   SECRET                      AGE
    # traefik     wildcard-fhuebung-lol   True    wildcard-fhuebung-lol-tls   ...
    ```

    The row must show `READY=True`. If it does not, do not start the demo —
    no new tenant URL will serve TLS until the wildcard is reissued.

5. **platform-gitops repo** on a clean `main` locally (you will branch off it
    in a moment):

    ```bash
    cd platform-gitops
    git fetch --quiet origin
    git checkout main
    git pull --ff-only
    ```

## Pre-staged artefacts (recommended)

Open the claim PR and collect one approval **before the demo starts**. This
removes commitlint, CI, and CODEOWNERS-approval risk from the live timeline.

From the `platform-gitops` repo root:

```bash
git checkout -b rl/feat/livedemoNN-tenant

cat > tenants/livedemoNN.yaml <<'EOF'
---
apiVersion: platform.fhuebung.lol/v1alpha1
kind: Tenant
metadata:
  name: livedemoNN
spec:
  name: livedemoNN
  tier: small
EOF

git add tenants/livedemoNN.yaml
git commit -m "feat(tenants): add livedemoNN tenant for live demo"
git push -u origin rl/feat/livedemoNN-tenant

gh pr create -R INENI-PT-GROUP-B/platform-gitops --draft \
  --title "feat(tenants): add livedemoNN tenant" \
  --body "Onboards a fresh demo tenant under https://livedemoNN.fhuebung.lol for a live demonstration of the end-to-end tenant onboarding flow. Will be offboarded after the demo."
```

Ping a teammate out-of-band to approve and **leave the PR in Draft** until
the demo. During the demo you click "Ready for review" and squash-merge in
one action.

**Live-write path (more ambitious, riskier).** Skip pre-staging and
demonstrate branch + commit + PR creation as part of the demo trail.
Allocate at least two extra minutes of slack and have a fallback narrative
ready if CI is slow or a check fails.

## The claim

The whole claim file is seven lines. The shape is enforced by the XRD at
`platform-gitops/crossplane/xrds/xtenant.yaml`.

```yaml
---
apiVersion: platform.fhuebung.lol/v1alpha1
kind: Tenant
metadata:
  name: livedemoNN
spec:
  name: livedemoNN
  tier: small
```

Note: `kind: Tenant` is the **claim** (what you write); Crossplane creates
an `XTenant` **composite** from it on reconciliation. Both are queryable.

Name constraints (RFC1123 lowercase label, ≤ 56 chars, `metadata.name` must
equal `spec.name`, reserved names like `staging` excluded) and tier-selection
policy live in
[`tenant-onboarding.md`](https://github.com/INENI-PT-GROUP-B/platform-gitops/blob/main/docs/tenant-onboarding.md#step-1--choose-a-name-and-a-tier).
For the demo `tier: small` is always the right pick.

## Demo trail (step-by-step)

Each step lists what the audience should see. If you see something else,
jump to **Fallback**.

### Step 1 — Show the claim diff

In a browser tab, open the PR (pre-staged or live-written). Click
**Files changed** and walk the audience through the seven-line claim.
Read out `spec.name` and `spec.tier`.

### Step 2 — Merge

Click **Squash and merge** (or **Ready for review** → **Squash and merge**
if the PR is in Draft). Confirm the squash subject reads sensibly.

Mention: "Argo CD picks this up on its next poll — every three minutes by
default. I'll trigger it manually now so we don't wait."

### Step 3 — Argo CD UI: trigger the sync

Switch to the Argo CD UI tab. Open the **`tenants`** Application. Click
**Sync** in the top right, leave defaults, confirm.

Expected: Application transitions `OutOfSync → Syncing → Synced`. The
resource tree expands to show the new `Tenant` claim and the `XTenant`
composite Crossplane created from it. Underneath the composite, the tree
shows the namespace, ResourceQuota, LimitRange, four NetworkPolicies,
the CloudNativePG `Cluster`, the BasicAuth chain (Password generator →
htpasswd `ExternalSecret` → GSM `SecretVersion` → Traefik `Middleware`),
the GHCR pull `ExternalSecret`, and the Helm `Release`.

### Step 4 — Terminal: claim and composite Ready

```bash
kubectl -n crossplane-system get tenant livedemoNN
# NAME         SYNCED   READY   CONNECTION-SECRET   AGE
# livedemoNN   True     True                        1m

kubectl get xtenant -l crossplane.io/claim-name=livedemoNN
# NAME                SYNCED   READY   COMPOSITION       AGE
# livedemoNN-<hash>   True     True    xtenant-default   1m

kubectl get managed -A | grep livedemoNN
# (rows for the provider-helm Release, provider-gcp Secret + SecretVersion,
#  BucketIAMMember, ExternalSecrets, Middleware, …)
```

### Step 5 — Tenant namespace workloads

```bash
kubectl -n tenant-livedemoNN get pods
# NAME                                  READY   STATUS    RESTARTS   AGE
# livedemoNN-backend-<rs>-<pod>         1/1     Running   0          1m
# livedemoNN-frontend-<rs>-<pod>        1/1     Running   0          1m
# livedemoNN-cluster-1                  1/1     Running   0          1m

kubectl -n tenant-livedemoNN get cluster
# NAME                  AGE   INSTANCES   READY   STATUS                     PRIMARY
# livedemoNN-cluster    1m    1           1       Cluster in healthy state   livedemoNN-cluster-1
```

### Step 6 — Browser: the app is live

Open <https://livedemoNN.fhuebung.lol>. Expected:

- Valid TLS (wildcard certificate covers the host — no browser warning).
- BasicAuth prompt. Username is `admin`. Two ways to retrieve the password
    depending on your GCP IAM:

    ```bash
    # Preferred path — requires roles/secretmanager.secretAccessor on the secret:
    gcloud secrets versions access latest \
      --secret="tenant-livedemoNN-basicauth-password"

    # Fallback via the in-cluster source Secret — works for any team member
    # with read access to namespace crossplane-system, no GCP IAM needed:
    kubectl -n crossplane-system get secret tenant-livedemoNN-htpasswd-source \
      -o jsonpath='{.data.password}' | base64 -d
    ```

- After login: the property-management app loads, version badge visible
    in the corner.

Add one property to confirm the CNPG backend is wired through. This is the
moment to pause and let the audience digest: **claim YAML in → fully
provisioned isolated tenant out, in single-digit minutes, with no manual
cluster step in between**.

## Timing expectation

| Phase                                                                          | Typical duration |
| ------------------------------------------------------------------------------ | ---------------- |
| Squash-merge → Argo CD picks up the manifest (after manual Sync click)         | ≤ 10 s           |
| Crossplane XR rendered, all output resources created                           | 10–30 s          |
| CNPG `Cluster` initialised (Pod scheduled, PostgreSQL up, backend Secret ready)| 60–120 s         |
| provider-helm Release rolled out (backend + frontend Pods ready)               | 30–90 s          |
| ExternalDNS publishes record, Cloud DNS propagates                             | 10–60 s          |
| **Total wall-clock, manual-sync accelerated**                                  | **3–6 min**      |

Without the manual Sync click, add up to three minutes for the next Argo CD
poll (`6–9 min`). The steady-state polled case is documented as 5–10 min in
[`tenant-onboarding.md`](https://github.com/INENI-PT-GROUP-B/platform-gitops/blob/main/docs/tenant-onboarding.md#what-happens-after-the-merge)
— our manual-sync acceleration cuts the lower bound.

## Fallback

### Argo CD sync looks stuck

Force a hard refresh via annotation patch:

```bash
kubectl -n argocd annotate application tenants \
  argocd.argoproj.io/refresh=hard --overwrite
```

### Crossplane reconcile not progressing

```bash
kubectl -n crossplane-system describe tenant livedemoNN
# Scroll to Events; provider connect / patch / WaitingForReady reasons.

kubectl describe xtenant -l crossplane.io/claim-name=livedemoNN
# Composite-level conditions show which managed resource is the laggard.
```

### BasicAuth prompt missing, 401 loop, or 502

```bash
kubectl -n tenant-livedemoNN get middleware
# → basicauth Middleware must exist

kubectl -n tenant-livedemoNN get externalsecret
# → status SecretSynced = True for both htpasswd and ghcr pull secret

kubectl -n tenant-livedemoNN get secret tenant-livedemoNN-basicauth \
  -o jsonpath='{.data.users}' | base64 -d
# → non-empty htpasswd content (admin:$2a$...)
```

### TLS warning in the browser

Wildcard cert is cluster-wide and persistent. A TLS warning almost always
means you opened `http://` instead of `https://`, or DNS has not propagated
yet. Hard-refresh and check:

```bash
dig +short livedemoNN.fhuebung.lol
```

### Worst case — anything blocks

Two layered fallbacks, in order:

1. **Switch to a pre-existing tenant.** Open <https://staging.fhuebung.lol>
    (or any other live tenant) and walk the audience through the same UI
    plus the verification commands above against a running namespace.
    Story still lands; only the live-creation moment is lost.
2. **Switch to the backup video.** A pre-recorded full walkthrough is
    captured during the S5-06 dry run (presentation-slot addendum, separate
    doc). Play it, narrate over.

## Cleanup (after the demo)

Offboard via PR — same motion as onboarding, in reverse. The full
offboarding semantics (what is deleted, what survives, timing) are in
[`tenant-onboarding.md` § Offboarding](https://github.com/INENI-PT-GROUP-B/platform-gitops/blob/main/docs/tenant-onboarding.md#offboarding).
Quick form, from the `platform-gitops` repo root:

```bash
git checkout main
git pull --ff-only
git checkout -b rl/chore/remove-livedemoNN
rm tenants/livedemoNN.yaml
git add tenants/livedemoNN.yaml
git commit -m "chore(tenants): remove livedemoNN after demo"
git push -u origin rl/chore/remove-livedemoNN

gh pr create -R INENI-PT-GROUP-B/platform-gitops \
  --title "chore(tenants): remove livedemoNN after demo" \
  --body "Offboards livedemoNN tenant after the live demonstration."
```

Once squash-merged, prune the local branch:

```bash
git checkout main
git pull --ff-only
git branch -d rl/chore/remove-livedemoNN
```

**Note on GSM secrets.** `tenant-livedemoNN-basicauth-htpasswd` and
`tenant-livedemoNN-basicauth-password` are deliberately orphaned by the
Composition's `deletionPolicy: Orphan` on the GSM resources. Re-using the
same name later restores the same secret history; pick a fresh name if
you want a fresh secret history. Full semantics: see
[`tenant-onboarding.md` § Offboarding](https://github.com/INENI-PT-GROUP-B/platform-gitops/blob/main/docs/tenant-onboarding.md#offboarding).

---

*Companion document for steady-state operations:*
[*`platform-gitops/docs/tenant-onboarding.md`*](https://github.com/INENI-PT-GROUP-B/platform-gitops/blob/main/docs/tenant-onboarding.md).
