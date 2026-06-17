# Capacity & Cost — Actuals vs Estimate

How well the S1-15 cost estimate (`capacity_costs_gcp.md`) held up against
the actual GCP billing for project `dotted-axle-495612-f4`, and what
drove the differences.

## Period

Billing window: **2026-05-07 to 2026-06-16 — 40 calendar days.** Inside
that window the cluster was actually running for **~18 days** (first
end-to-end bootstrap on 2026-05-30 through the S4-04b teardown on
2026-06-15, plus the fresh cluster from 2026-06-15 19:08 UTC onwards).
Cluster-active services are normalised to a per-active-month rate
(cost × 30 / 18) so they compare with the estimate's per-month figures.
Cloud DNS and the Terraform state bucket bill for the full 40 days
regardless of cluster state.

Source data is in the `actuals/` folder:

- `Report_HAW_Billing_Account_Google Cloud Console_05+06-2026.pdf`
- `Details_HAW_Billing_Account_2026-05-07 - 2026-06-16.csv`
- A June-only slice (`..._06-2026.pdf` +
  `..._2026-06-01 - 2026-06-16.csv`) for reference.

## Summary

| | Estimate | Actual (per-month equivalent) | Granted budget |
| --- | --- | --- | --- |
| Per month | 441.27 € | ~360 € | 500 € |
| Actually billed for the 40-day window | — | ~216 € | — |

We came in roughly **18 % under the estimate** and **30 % under the
granted budget**. The estimate held its overall shape; the deltas come
from two clear places, one in our favour and one against — see the
notes after the table.

These figures apply to the 40-day window through 2026-06-13 where the
K8s Engine credit was active. From 2026-06-14 the credit no longer
applies — see § Forecast through submission for the post-window
picture.

## Per-category comparison

| Category | Estimate / month | Actual / month | Reason |
| --- | --- | --- | --- |
| Compute Engine (worker nodes, boot disks, persistent disk storage) | 329.85 € | ~313 € | The cluster autoscaler is set to 3–6 worker nodes. It stayed at the 3-node minimum the whole window, which is exactly what the estimate assumed — so the numbers match almost 1:1. |
| Kubernetes Engine (cluster management fee) | 62.42 € | ~7 € | Google charges a flat monthly fee for the control plane of every Kubernetes cluster (~70 €/month for a zonal cluster). GCP's "Always Free" tier provides a monthly credit (~70 €) for this fee per *billing account* — shared across all projects on it. The HAW educational billing account (`01870E-E5B717-2F48D1`) hosts multiple projects; the shared monthly cap absorbed our cluster's management fee through 2026-06-13, which is what the ~7 € rate above reflects. From 2026-06-14 the shared cap for June was reached and our cluster now pays the full management fee — the daily breakdown for that line on the Console no longer shows an offsetting credit. For the remainder of June through the submission deadline the cluster pays the full management fee; the cost forecast through submission below factors that in. |
| Networking (Load Balancer, NAT Gateway, egress) | 23.41 € | ~15 € | The estimate modelled 50 GiB of monthly outbound traffic; demo + validation usage came in well under that. The first GB of egress per region per month is also part of the "Always Free" allowance. |
| Cloud DNS (managed zone + lookups) | 0.86 € | 0.16 € | Cloud DNS bills per DNS lookup. Every browser hit on a `*.fhuebung.lol` URL, every Let's Encrypt cert check, every ExternalDNS update sends one. The estimate planned for 2 M lookups / month assuming some public demo traffic; actual usage was around 400 k over 40 days, dominated by internal traffic (cert checks, ExternalDNS, manual validation). |
| Secret Manager (passwords, tokens, GHCR PAT) | 2.49 € | <0.10 € | The "Always Free" tier covers up to 6 active secret versions and 10 000 access operations per month. We stayed under both. |
| Cloud Logging (cluster + application logs) | 22.23 € | 0 € | The first 50 GiB of logs per month are part of "Always Free". The cluster is small and the apps don't log much, so we never came close. |
| Cloud Monitoring (metrics, dashboards) | 0 € | ~25 € | GKE was created with "Google Managed Service for Prometheus" enabled. That GKE feature automatically forwards detailed cluster metrics (CPU, memory, pod health, container health) to Cloud Monitoring, on top of the in-cluster Prometheus + Grafana stack we already run. The estimate assumed only the free GKE metrics would be billed; the richer GKE-side path costs a few euro per month. |
| Cloud Storage (Terraform state bucket) | 0.02 € | 0 € | 1 GiB of storage stays inside the "Always Free" allowance. |
| **Total** | **441.27 €** | **~360 €** | **−18 % overall. Biggest positive: "Always Free" on the GKE control-plane fee (~55 €/month). Biggest negative: Cloud Monitoring via Managed Prometheus (~25 €/month), not modelled in the estimate. Everything else came in close to plan or under it.** |

## Why we paid less than planned

- **GKE control-plane fee fell under "Always Free".** Single biggest
  saving. The CSV shows a -41.59 € "Andere Einsparungen" line booked
  against Kubernetes Engine over the 40-day window — that's the full
  cluster management fee for the cluster-active days.
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
  Grafana stack. ~25 €/month while the cluster is active. The estimate
  assumed the active monitoring path would be in-cluster only, so this
  line wasn't modelled. Worth flagging for the scaling discussion —
  this cost grows roughly with the number of namespaces, so on a 100-
  or 1000-tenant fleet it would become non-trivial.

## Forecast through submission (2026-06-26)

The cluster runs another ~9 days through the submission deadline.
With the K8s Engine credit no longer applying for June from
2026-06-14 (see the K8s row above), the management fee accrues at
the full ~$2.50 USD/day. Estimated additional charges from
2026-06-17 through 2026-06-26:

- K8s Engine management fee: ~9 days × ~2.30 €/day ≈ 21 €
- Compute Engine (worker nodes + boot disks + PD):
  ~9 days × ~10.40 €/day ≈ 94 €
- Cloud Monitoring (GMP): ~9 days × ~0.85 €/day ≈ 8 €
- Networking, DNS, Logging: ~5 €.

Estimated additional bill through deadline: **~130 €.**

## Budget headroom

| | Per month | Sprint 1–4 (≈2 months) |
| --- | --- | --- |
| Granted budget | 500 € | 1000 € |
| Estimate (S1-15) | 441.27 € | 882.54 € (requested credit envelope) |
| Actual (per-month equivalent) | ~360 € | ~720 € extrapolated |
| Actually billed | — | ≈216 € for the 40-day window |

Roughly 80 €/month headroom under the estimate, 140 €/month under the
granted budget. The Sprint 1–4 window came in well inside the credit
envelope from S1-15.

## References

- Estimate: [`capacity_costs_gcp.md`](./capacity_costs_gcp.md) (S1-15,
  2026-05-03).
- Source data: [`actuals/`](./actuals/).
- Cluster-redeploy validation that shaped the billing window:
  [`platform-gitops/docs/cluster-redeploy-validation.md`](https://github.com/INENI-PT-GROUP-B/platform-gitops/blob/main/docs/cluster-redeploy-validation.md)
  (S4-04b, `platform-gitops#65`).
- Issue: `INENI-PT-GROUP-B/platform#100`.
- Feeds S5-04 (capacity actuals + scaling forecast slides).
