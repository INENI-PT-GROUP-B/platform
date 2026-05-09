# Capacity & Cost Planning

## Project Setup

- **Region:** europe-west1 (Belgium)
- **Setup:** GKE Standard, zonal, 3 Worker Nodes
- **Hardware per Worker Node:** 4 vCPU, 16 GB RAM, 100 GiB SSD
- **Cluster total for Worker Nodes:** 12 vCPU, 48 GB RAM, 300 GiB SSD

## Service Breakdown with Costs

### Compute Resources

#### GKE (Kubernetes Engine): 382.10 EUR/month

| Item | Cost | Purpose |
|---|---|---|
| Zonal Cluster Management Fee | 62.42 EUR | Managed Control Plane for GKE |
| Boot Disks SSD persistent 3x100 GiB | 43.61 EUR | OS and container image cache for the 3 worker nodes |
| E2 Instances 3x e2-standard-4 | 276.07 EUR | Worker nodes for platform components and tenants |

#### Persistent Disk Workload Storage: 10.18 EUR/month

| Item | Cost | Purpose |
|---|---|---|
| Zonal SSD PD 70 GiB | 10.18 EUR | Persistent volumes for monitoring (Prometheus, Grafana) and application database (CloudNativePG) |

### Networking

#### Cloud Load Balancing: 15.74 EUR/month

| Item | Cost | Purpose |
|---|---|---|
| 1 Forwarding Rule Regional External Load Balancer | 15.60 EUR | Central HTTPS entry point for all platform and tenant endpoints |
| Inbound Data 10 GiB | 0.07 EUR | Incoming user traffic |
| Outbound Data 10 GiB | 0.07 EUR | API responses and asset delivery |

#### Cloud NAT Gateway: 7.67 EUR/month

| Item | Cost | Purpose |
|---|---|---|
| Gateway Uptime for 3 VM instances | 2.62 EUR | Egress routing for private worker nodes |
| Data Processing 50 GiB | 1.92 EUR | Egress volume from the cluster |
| IP Address | 3.12 EUR | Static public IP for the NAT gateway |

### DNS and Secrets

#### Cloud DNS: 0.86 EUR/month

| Item | Cost | Purpose |
|---|---|---|
| 1 Managed Zone | 0.17 EUR | Management of the platform domain |
| DNS Queries 2M | 0.68 EUR | DNS resolution for platform endpoints and certificate challenges |

#### Secret Manager: 2.49 EUR/month

| Item | Cost | Purpose |
|---|---|---|
| 50 Active Secret Versions | 2.26 EUR | Storage of all platform and tenant credentials |
| 100,000 Access Operations | 0.23 EUR | Synchronization of secrets into the cluster |

### Operations

#### Cloud Logging: 22.23 EUR/month

| Item | Cost | Purpose |
|---|---|---|
| Log Storage 100 GiB | 21.38 EUR | Aggregation and analysis of cluster and application logs |
| Log Retention 1 Month | 0.86 EUR | Standard retention for troubleshooting and audit |

#### Cloud Monitoring: 0.00 EUR/month

| Item | Cost | Purpose |
|---|---|---|
| Standard GKE Metrics | 0.00 EUR | Passive health monitoring, active monitoring runs via in-cluster Prometheus and Grafana |

### Storage

#### Cloud Storage: 0.02 EUR/month

| Item | Cost | Purpose |
|---|---|---|
| 1 GiB Standard Storage | 0.02 EUR | Terraform remote state backend with versioning and state locking |

## Total Costs

| Category | EUR/month |
|---|---|
| Compute (GKE + Persistent Disk) | 392.27 |
| Networking (Load Balancer + NAT) | 23.41 |
| DNS + Secrets | 3.35 |
| Operations (Logging + Monitoring) | 22.23 |
| Storage (Terraform State) | 0.02 |
| **Total** | **441.27 EUR/month** |


## Credit Request Calculation

- Active cluster operation: approximately 2 months (starting from May until end of June 2026)
- Base: 441.27 EUR x 2 months = 882.54 EUR
- Recommended request: **882.54 EUR**

## Calculator References

- GCP Pricing Calculator Estimate: [GCP Pricing Calculator Estimate](https://cloud.google.com/products/calculator/estimate-preview/CiRjMzUzNDcyOC1lY2YwLTQyMDYtYmY4OC0wYWYzNGJkMWQ1N2IQAQ%3D%3D?_gl=1*xdoyu3*_up*MQ..&gclid=CjwKCAjw5NvPBhAoEiwA_2egfgXL06KlOBHH5otEKWucYJzMzgLPsH0-2LzR07sjdGNJb911TGlw0xoCShoQAvD_BwE&gclsrc=aw.ds)
- CSV export with all SKUs as evidence [calculation_gcp_03052026.csv](https://github.com/INENI-PT-GROUP-B/platform/blob/main/docs/cost-planning/calculation_gcp_03052026.csv)
- PDF summary as attachment [Google_Cloud_Estimate_Summary_03052026.pdf](https://github.com/INENI-PT-GROUP-B/platform/blob/main/docs/cost-planning/Google_Cloud_Estimate_Summary_03052026.pdf)