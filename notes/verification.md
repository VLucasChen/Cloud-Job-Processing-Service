# Fact Verification Log (9 Oct 2026)

Every external fact used in the report, the value used, where it was checked, and how
strong the evidence is. Workload figures (stage times, prep output size, report size) are
**design assumptions**, not facts, and are labelled as such in the report.

Status: **Official** = provider/project documentation · **Third-party** = price tracker or
blog · **Partial** = official but old, or only partly confirms the claim · **Unverified** =
worded cautiously in the report.

## Networking

| Claim | Value used | Source | Status |
|---|---|---|---|
| AWS Site-to-Site VPN bandwidth per standard tunnel | ≤ 1.25 Gbps; ECMP across tunnels needs Transit Gateway; 5 Gbps "large" tunnels only on TGW/Cloud WAN | [AWS VPN quotas](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-limits.html), [Tunnel options](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPNTunnels.html) | Official |
| GCP HA VPN ↔ AWS topology | 2 AWS VPN connections → 4 tunnels; 4 tunnels needed for 99.99% SLA | [Create HA VPN between Google Cloud and AWS](https://docs.cloud.google.com/network-connectivity/docs/vpn/tutorials/create-ha-vpn-connections-google-cloud-aws) | Official |
| Same-metro RTT AWS Sydney ↔ GCP Sydney | < 5 ms | — | Unverified → "to be measured" |
| Upload burst bandwidth | 100 × 3 GB / 3600 s = 0.667 Gbps | arithmetic | Checked |

## Databases and coordination

| Claim | Value used | Source | Status |
|---|---|---|---|
| Patroni synchronous mode | `/sync` key holds leader + sync standby; a node not in `/sync` cannot auto-promote; a member stays sync in PostgreSQL while `/sync` lists it | [Patroni replication modes](https://patroni.readthedocs.io/en/latest/replication_modes.html) | Official |
| Removal ordering (`/sync` updated before leader stops waiting) | follows from the documented invariant; release notes say the two are updated "in a very specific order" | [Patroni release notes](https://patroni.readthedocs.io/en/master/releases.html) | Partial |
| Cancelled commit waits | a cancelled wait leaves a locally committed, unreplicated transaction → report forbids cancelling `COMMIT` | Patroni docs + [PostgreSQL syncrep discussion](https://www.postgresql.org/message-id/cac4b9df-92c6-77aa-687b-18b86cb13728@stratox.cz) | Official |
| Patroni watchdog and timing | defaults `ttl=30`, `loop_wait=10`; watchdog fires 5 s before TTL; primary demotes when leader key cannot be renewed | [Patroni watchdog](https://patroni.readthedocs.io/en/latest/watchdog.html) | Official |
| etcd fault tolerance | 3 members → quorum 2, tolerates 1 failure | [etcd FAQ](https://etcd.io/docs/v3.5/faq/) | Official |
| PostgreSQL `now()` vs `clock_timestamp()` | `now()` = transaction start; `clock_timestamp()` = actual time, changes during a statement | [PostgreSQL date/time functions](https://www.postgresql.org/docs/current/functions-datetime.html) (confirmed via secondary sources quoting it) | Official (indirect) |
| `SELECT … FOR UPDATE SKIP LOCKED`, row-level security | standard PostgreSQL features | PostgreSQL docs | Official |
| Managed DBs cannot form a cross-provider synchronous pair | RDS/Cloud SQL HA is in-provider | general knowledge, not found in docs | Unverified → reworded as "managed DB (in-provider HA)" |

## Storage, identity, orchestration

| Claim | Value used | Source | Status |
|---|---|---|---|
| S3 create-only writes | `If-None-Match` on PutObject / CompleteMultipartUpload | [S3 conditional writes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html) | Official |
| GCS create-only writes | `ifGenerationMatch=0` → 412 if object exists | [GCS request preconditions](https://docs.cloud.google.com/storage/docs/request-preconditions) | Official |
| AWS STS minimum session | `DurationSeconds` min 900 s (15 min) | [AssumeRole API](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) | Official |
| EKS node-group update strategy | `updateStrategy: DEFAULT \| MINIMAL`; MINIMAL terminates old nodes before launching new | [NodegroupUpdateConfig](https://docs.aws.amazon.com/eks/latest/APIReference/API_NodegroupUpdateConfig.html) | Official |
| GKE surge settings | `maxSurge` / `maxUnavailable` per zone; `maxSurge=0, maxUnavailable=1` avoids extra nodes | [GKE node pool upgrade strategies](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/node-pool-upgrade-strategies) | Official |
| KEDA ScaledJob | `accurate` strategy subtracts pending jobs | [KEDA ScaledJob spec](https://keda.sh/docs/2.16/reference/scaledjob-spec/) | Official |

## GPUs (changed the design)

| Claim | Value used | Source | Status |
|---|---|---|---|
| GCP Sydney GPUs | T4 in `australia-southeast1-a` and `-c`; P4, P100; A3 Mega limited; **no L4/G2**; Melbourne no GPUs | [GCP GPU regions and zones](https://cloud.google.com/compute/docs/gpus/gpu-regions-zones) | Official (confirm with `gcloud compute accelerator-types list`) |
| AWS Sydney T4 | G4 (`g4dn`, NVIDIA T4 16 GB) launched in Sydney in 2019; capacity shortages have been reported | [AWS announcement](https://aws.amazon.com/about-aws/whats-new/2019/10/amazon-ec2-g4-instances-with-nvidia-t4-tensor-core-gpus-now-available-in-6-additional-regions/), [re:Post](https://repost.aws/questions/QUgdskPRUYQRWbxnO2VoIzMw/g4-and-g5-instances-not-available-in-sydney-region-for-multiple-days) | Official |
| **Design change** | both clouds use T4 (was "L4 preferred"); model assumed to fit 16 GB VRAM | — | — |

## Prices (planning figures)

| Item | Value used | Source | Status |
|---|---|---|---|
| AWS egress Sydney → internet | ≈ $0.114/GB (first 10 TB) | [AWS 2018 Australia price cut](https://aws.amazon.com/blogs/aws/aws-data-transfer-price-reductions-up-to-34-japan-and-28-australia/), DoiT table | Partial (old official + third-party) |
| GCP Premium Tier egress to Australia | $0.19/GiB (0–1 TiB), $0.18 (1–10 TiB) | [GCP network pricing](https://cloud.google.com/vpc/network-pricing) | Partial (page may be dated) |
| AWS `g4dn.xlarge` Sydney | ≈ $0.684/h on-demand | [DoiT](https://www.doit.com/compute/spot/ap-southeast-2/g4dn.xlarge), Devzero | Third-party |
| GCP `n1-standard-4` Sydney | ≈ $0.27/h | [Holori](https://calculator.holori.com/gcp/vm/n1-standard-4/australia-southeast1) | Third-party |
| GCP T4 GPU | $0.35/h US list; Sydney rate not found | [GCP GPU pricing](https://cloud.google.com/products/compute/gpus-pricing) | Partial |
| S3 Standard Sydney | ≈ $0.025/GB-month (first 50 TB) | regional comparison + user report citing AWS page | Third-party |

## Recomputed numbers

- GPU always-on: 4 × 720 h × $0.70 = **$2,016/month**.
- GPU on demand: 10 GPU-h/day × 30 × $0.70 = $210, plus 1 warm GPU 12 h/day = $252 → **≈ $462/month**; saving ≈ $1.55k.
- Mirroring all inputs: 9 TB/month → 9,000 GB × $0.114 = $1,026 (AWS) to 8,382 GiB tiered at $0.19/$0.18 ≈ $1,519 (GCP) → **≈ $1.0–1.5k/month**.
- One prepared-data spill: 0.15–0.3 GB × $0.114–0.19 → **≈ $0.02–0.05**.
- Report mirroring: 100 × 10 MB × 30 = 30 GB/month → ≈ $3–6.
