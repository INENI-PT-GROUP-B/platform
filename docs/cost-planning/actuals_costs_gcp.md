# Capacity & Cost — Actuals vs Estimate

How well the S1-15 cost estimate (`capacity_costs_gcp.md`) held up against
the actual GCP billing for project `dotted-axle-495612-f4`, and what
drove the differences.

## Period

The per-category actuals below reflect the **steady-state daily rate**
from the GCP Billing Console snapshot on 2026-06-20 — the first full
week without the K8s "Always Free" credit — extrapolated to a monthly
figure as **daily rate × 30**. Every line is directly verifiable
against the Console daily view for that date (14.11 €/day net total).
The pre-14.06. credited-window phase is reported separately in the
Summary section: same per-service rates minus the K8s control-plane
fee (absorbed by the credit through 2026-06-13), totalling ~361 €/month.

Source data is in the `actuals/` folder:

- **Methodology anchor** (2026-06-20 snapshot, all per-service rates
  derived from this):
  - `Report_HAW_Billing_Account_Google Cloud Console_06-2026-Day20.pdf`
  - `Details_HAW_Billing_Account_2026-06-20.csv`
- **Billing context** (for cross-checks against the broader window):
  - `Report_HAW_Billing_Account_Google Cloud Console_05+06-2026.pdf`
  - `Report_HAW_Billing_Account_Google Cloud Console_06-2026.pdf`
  - `Details_HAW_Billing_Account_2026-05-07 - 2026-06-20.csv`
  - `Details_HAW_Billing_Account_2026-06-01 - 2026-06-30.csv`

## Summary

| | Per month |
| --- | --- |
| Estimate (S1-15) | 441 € |
| Granted budget | 500 € |
| Actual rate through 2026-06-13 (with K8s "Always Free" credit) | ~361 € |
| Steady-state rate from 2026-06-14 (full K8s management fee) | ~423 € |

For the window where the K8s Engine credit applied (through
2026-06-13) we ran **~18 % under the estimate** and **~28 % under the
granted budget**. From 2026-06-14 the shared "Always Free" cap on the
billing account was exhausted (see § Forecast). The steady-state rate
from then on is **~423 €/month** — roughly on plan with the estimate
(~4 % under) and ~15 % under the granted budget. The estimate held
its overall shape; the deltas come from two clear places, one in our
favour and one against — see the notes after the table.

## Per-category comparison

| Category | Estimate / month | Actual / month | Reason |
| --- | --- | --- | --- |
| Compute Engine (worker nodes, boot disks, persistent disk storage) | 329.85 € | ~315 € | The cluster autoscaler is set to 3–6 worker nodes. It stayed at the 3-node minimum the whole window, which is exactly what the estimate assumed — so the numbers match almost 1:1. |
| Kubernetes Engine (cluster management fee) | 62.42 € | ~62 € | Google charges a flat monthly fee for the control plane of every Kubernetes cluster. For our zonal cluster in europe-west1 this lands at ~62 €/month (2.06 €/day, verifiable on the GCP Console from 14.06. onwards) — the ~62 € rate above reflects this steady-state. GCP's "Always Free" tier provides a monthly credit (~70 €) for this fee per *billing account* — shared across all projects on it. The HAW educational billing account (`01870E-E5B717-2F48D1`) hosts multiple projects; the shared monthly cap absorbed our cluster's management fee through 2026-06-13 (effectively 0 €/month during that phase). From 2026-06-14 the shared cap for June was reached — the daily breakdown for that line on the Console no longer shows an offsetting credit. For the remainder of June through the submission deadline the cluster pays the full management fee. |
| Networking (Load Balancer, NAT Gateway, egress) | 23.41 € | ~16 € | The estimate modelled 50 GiB of monthly outbound traffic; demo + validation usage came in well under that. The first GB of egress per region per month is also part of the "Always Free" allowance. |
| Cloud DNS (managed zone + lookups) | 0.86 € | ~0.30 € | Cloud DNS bills per DNS lookup. Every browser hit on a `*.fhuebung.lol` URL, every Let's Encrypt cert check, every ExternalDNS update sends one. The estimate planned for 2 M lookups / month assuming some public demo traffic; actual steady-state rate is 0.01 €/day (~0.30 €/month), dominated by internal traffic (cert checks, ExternalDNS, manual validation) with no public demo lookups. |
| Secret Manager (passwords, tokens, GHCR PAT) | 2.49 € | ~1.20 € | The "Always Free" tier covers up to 6 active secret versions and 10 000 access operations per month. Daily rate is 0.04 €/day (~1.20 €/month) — slightly above the free tier on access operations, but well below the 2.49 € modelled in the estimate. |
| Cloud Logging (cluster + application logs) | 22.23 € | 0 € | The first 50 GiB of logs per month are part of "Always Free". The cluster is small and the apps don't log much, so we never came close. |
| Cloud Monitoring (metrics, dashboards) | 0 € | ~29 € | GKE Standard auto-enables Google Managed Service for Prometheus by default (`managedPrometheusConfig.enabled = true`). That feature forwards cluster metrics into Cloud Monitoring as managed time-series — a parallel pipeline to our in-cluster Prometheus + Grafana stack, which is what we actually use for monitoring. We do not consume the default pipeline, but GKE bills it regardless. Daily rate 0.98 €/day (~29 €/month). |
| Cloud Storage (Terraform state bucket) | 0.02 € | 0 € | 1 GiB of storage stays inside the "Always Free" allowance. |
| **Total** | **441.27 €** | **~423 €** | **−4 % overall at steady-state (from 14.06.). The credited-window phase ran at ~361 €/month (−18 % vs. estimate). Biggest positive (now ended): "Always Free" credit absorbed the GKE control-plane fee — ~62 €/month while it applied, through 2026-06-13. Biggest negative (ongoing): Cloud Monitoring via Google Managed Prometheus (~29 €/month), enabled by default in GKE Standard, not modelled in the estimate. Everything else came in close to plan or under it.** |

## Why we paid less than planned

- **GKE control-plane fee fell under "Always Free".** Single biggest
  saving for the credited window. Through 2026-06-13 the shared
  Always Free credit on the billing account absorbed the full cluster
  management fee. From 2026-06-14 the cap was exhausted; the cluster
  now pays the full ~62 €/month management fee (2.06 €/day) — see
  § Forecast for what that means going forward.
- **DNS lookup volume was ~5× lower than modelled.** The 2 M / month
  estimate assumed external visitors on the demo URLs; in practice
  traffic was almost entirely internal (cert checks, ExternalDNS,
  manual validation).
- **Log volume stayed under the 50 GiB free cap.** A small platform plus
  four single-replica tenant apps don't generate enough log volume to
  cross the free-tier ceiling.

## Why we paid more than planned

- **Cloud Monitoring via Managed Prometheus.** GKE Standard now creates
  clusters with `managedPrometheusConfig.enabled = true` by default;
  that feature forwards GKE cluster metrics into Cloud Monitoring as
  managed time-series, separate from our in-cluster Prometheus +
  Grafana stack. ~29 €/month at steady-state. The estimate
  assumed the active monitoring path would be in-cluster only, so this
  line wasn't modelled. Worth flagging for the scaling discussion —
  this cost grows roughly with the number of namespaces, so on a 100-
  or 1000-tenant fleet it would become non-trivial.

## Forecast through submission (2026-06-26)

The cluster runs another ~9 days through the submission deadline.
With the K8s Engine credit no longer applying for June from
2026-06-14 (see the K8s row above), the management fee accrues at
the full ~$2.50 USD/day. Estimated additional charges from 2026-06-17 through 2026-06-26 (daily rates from GCP Console snapshot 2026-06-20):

- K8s Engine management fee: ~9 days × 2.06 €/day ≈ 19 €
- Compute Engine (worker nodes + boot disks + PD):
  ~9 days × 10.50 €/day ≈ 95 €
- Cloud Monitoring (GMP): ~9 days × 0.98 €/day ≈ 9 €
- Networking, DNS, Secret Manager, Logging: ~5 €.

Estimated additional bill through deadline: **~128 €.**

That works out to a steady-state per-month rate of **~423 €/month** —
roughly on plan with the 441 €/month estimate (~4 % under) and
~15 % under the 500 €/month granted budget. The −18 % undershoot
shown in § Summary was driven by the K8s Engine credit; once that
stopped, the actual rate moved very close to the estimate.

## Budget headroom

| | Per month | Sprint 1–4 (≈2 months) |
| --- | --- | --- |
| Granted budget | 500 € | 1000 € |
| Estimate (S1-15) | 441.27 € | 882.54 € (requested credit envelope) |
| Actual through 2026-06-13 (credited) | ~361 € | ~722 € extrapolated |
| From 2026-06-14 (full K8s fee) | ~423 € | ~846 € extrapolated |

For the credited window we ran ~80 €/month under the estimate and
~139 €/month under the granted budget. From 2026-06-14 onwards the
rate moves closer to the estimate — still ~18 €/month under the
estimate and ~77 €/month under the granted budget. The Sprint 1–4
window came in well inside the credit envelope from S1-15.

## References

- Estimate: [`capacity_costs_gcp.md`](./capacity_costs_gcp.md) (S1-15,
  2026-05-03).
- Source data: [`actuals/`](./actuals/).
- Cluster-redeploy validation that shaped the billing window:
  [`platform-gitops/docs/cluster-redeploy-validation.md`](https://github.com/INENI-PT-GROUP-B/platform-gitops/blob/main/docs/cluster-redeploy-validation.md)
  (S4-04b, `platform-gitops#65`).
- Issue: `INENI-PT-GROUP-B/platform#100`.
- Feeds S5-04 (capacity actuals + scaling forecast slides).
