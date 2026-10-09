# Assumptions and Sizing (final)

Planning figures behind the report's assumptions table. Workload numbers are design hypotheses;
prices are planning figures, not quotes. Sources and evidence levels are in `verification.md`.

## 1. Workload

| Item | Given / assumed | Planning figure |
|---|---|---|
| Daily volume | given | 100 jobs/day |
| Burst | given | 100 submissions in 1 hour |
| Batch size | given 1–5 GB | mean **3 GB** (≈ 1,000 JPEGs) |
| Processing time | given 5–30 min | mean **15 min**, split **5 / 6 / 4 min** (prep / ID / report) |
| Prep output | assumed | ≈ 5–10% of input (150–300 MB) |
| Report | assumed | ≈ 10 MB |
| Model | assumed | fits a 16 GB T4 |

### GPU is the bottleneck
- 4 T4 GPUs × 60 / 6 min = **40 jobs/h**.
- A 100-job burst needs ≥ **2.5 h** of GPU service plus pipeline fill and cold start; ≈ 60 jobs still
  wait at the end of the burst hour → durable queue with ETA; acceptance never blocks on capacity.
- Daily GPU use ≈ 100 × 6 min = 10 GPU-h of 96 (≈ 10%) → GPU pools scale 0–2 per cloud.

### CPU at burst
- Prep: 100 × 5 min ≈ **8.3 VMs**; report is GPU-bound: 40 × 4 min ≈ **2.7 VMs** → ≈ 11 of 14 CPU workers.

## 2. VM budget (hard cap 20 VMs, ≤ 4 GPU)

| Pool | AWS | GCP | Total |
|---|---|---|---|
| System/state node (KEDA, CoreDNS, PostgreSQL + Patroni, etcd voter) | 1 | 1 | 2 |
| CPU workers (spot) | 0–7 | 0–7 | ≤ 14 |
| GPU workers (on-demand T4) | 0–2 | 0–2 | ≤ 4 |
| **Per-cloud cap** | **10** | **10** | **20** |

- Managed services (EKS/GKE masters, Fargate/Cloud Run, object storage) are assumed outside the cap.
- Upgrades must not exceed the cap: EKS `MINIMAL` update strategy, GKE `maxSurge=0, maxUnavailable=1`.
- An unreachable VM still counts. After a cloud loss the survivor keeps 10 VMs until the other cloud's
  VMs are confirmed terminated through its control plane; then CI raises limits up to 20 VMs / 4 GPUs
  (T4 quota pre-requested in both clouds).
- Rebalancing between clouds is allowed only while both are reachable.

## 3. Network

| Path | Assumption |
|---|---|
| Partner → object storage | burst 300 GB/h ≈ 670 Mbps (1.1 Gbps if every batch is 5 GB); direct multipart upload to presigned URLs |
| VM ↔ object storage | ≥ 5 Gbps |
| AWS ↔ GCP | 4 IPsec tunnels, ≤ 1.25 Gbps each (ECMP needs AWS Transit Gateway); plan on 1 Gbps effective |
| RTT AWS Sydney ↔ GCP Sydney | < 5 ms (to be measured); a synchronous commit costs a few ms |

## 4. Prices (planning figures)

| Item | Figure |
|---|---|
| Internet egress | ≈ $0.114/GB from AWS Sydney to $0.19/GiB for GCP Premium Tier to Australia |
| T4 GPU VM | ≈ $0.7/h (AWS `g4dn.xlarge` ≈ $0.68/h; GCP `n1-standard-4` ≈ $0.27/h + T4, $0.35/h US list) |
| S3 Standard Sydney | ≈ $0.025/GB-month |
| Fargate (us-east-1) | $0.04048/vCPU-h + $0.004445/GB-h → 0.5 vCPU / 1 GB task ≈ $18/month |

### Derived costs
- GPUs always-on: 4 × 720 h × $0.7 ≈ **$2.0k/month**; on demand: 10 GPU-h/day × 30 × $0.7 ≈ $210 plus
  1 warm GPU 12 h/day ≈ $252 → **≈ $0.46k/month** (saving ≈ $1.5k), for a ≈ 5 min cold start.
- Mirroring every raw input: 9 TB/month → **≈ $1.0–1.5k/month**; data-local placement avoids it.
- One prepared-data spill: 0.15–0.3 GB → ≈ $0.02–0.05.
- Report copies: 100 × 10 MB × 30 = 30 GB/month ≈ $3–6.
- Job service, two Fargate tasks: ≈ $40–45/month (Sydney somewhat above us-east-1).

## 5. Storage and retention

| Data | Store | Lifecycle |
|---|---|---|
| Inputs | per-tenant bucket + key, home cloud | versioned; deleted 30 days after the job ends; not replicated cross-cloud |
| Intermediates | `stage{n}/attempt-{uuid}/`, create-only | unpublished attempts GC'd after 24 h; published ones 7 days after the job ends |
| Reports | home cloud + copy in the other cloud | verified before `COMPLETED`; versioned; 1 year by policy; Infrequent Access after 30 days |
| Metadata | PostgreSQL + Patroni, one node per cloud | synchronous standby; WAL archived to S3 and GCS; PITR 14 days |

## 6. Third parties

| Dependency | If unavailable |
|---|---|
| Partner OIDC IdP | new logins fail; existing tokens keep working; job services validate JWTs locally with cached JWKS |
| Provider-neutral DNS with health checks | partners use per-cloud fallback hostnames |
| etcd quorum witness (third site, leader keys only) | AWS + GCP still form a quorum; witness + one cloud down → read-only |
| GitHub Actions (CI/CD, Terraform runs) | deployments pause; running system unaffected |
| Registries (ECR + Artifact Registry, mirrored) | each cloud pulls from its local registry |
