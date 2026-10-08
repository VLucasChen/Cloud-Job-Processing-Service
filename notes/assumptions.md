# Assumptions and Sizing (draft v0.1)

Working notes behind the "Assumptions" part of the report. Every number here
is a planning figure, not a quote. Where a figure drives a design choice, the
choice is named in **bold**. Items marked `[CONFIRM]` still need your decision.

Clouds used throughout: **AWS (primary DB) + GCP**, both in Sydney
(`ap-southeast-2`, `australia-southeast1`).

- Sydney rather than Melbourne: it is the larger region for both providers, with the
  widest GPU instance choice. Users submit batch jobs of 5–30 min, so the ~12 ms
  Melbourne–Sydney RTT does not matter.
- Same metro for both clouds → low inter-cloud latency; all data stays in Australia.
  This matters because camera traps also photograph people, so images may be personal
  information under the Privacy Act.
- Considered and rejected: one cloud in Sydney and one in Melbourne (better geographic
  independence, but fewer GPU options in Melbourne and higher inter-cloud latency).

---

## 1. Workload

| Item | Given / assumed | Planning figure |
|---|---|---|
| Daily volume | given | 100 jobs/day |
| Burst | given | 100 submissions in 1 hour (i.e. a whole day's load can arrive at once) |
| Batch size | given 1–5 GB | mean **3 GB**, about 1,000 JPEGs at ~3 MB each |
| Processing time | given 5–30 min, excluding queueing | mean **15 min** |
| Stage split (assumed) | prep / identify / report | **5 / 6 / 4 min** (GPU about 40%) |
| Prep output size (assumed) | resized and normalised images | **~5–10% of input** (~150–300 MB) `[CONFIRM: depends on supplied program]` |
| Report size | assumed | ~10 MB (PDF + CSV) |

### 1.1 GPU is the bottleneck

- GPU throughput = 4 GPUs × (60 / 6 min) = **40 jobs/hour**.
- Burst of 100 jobs → at least **2.5 h** of GPU service (100 / 40), plus pipeline fill and
  cold start; about 60 jobs still wait at the end of the burst hour. (v1 also claimed a
  5 h worst case; dropped in v2, because a 30-min job does not tell us its GPU share.)
- → **Durable queue with visible queue position/ETA**. Acceptance never blocks
  on capacity.
- Daily GPU demand = 100 × 6 min = 10 GPU-hours out of 96 available, so utilisation is about **10%**.
- → **GPU pool scales to demand** (min 0–1 warm, max 4). Always-on is not worth it (see §3).

### 1.2 CPU sizing at burst

- Prep can run ahead of the GPU: 100 × 5 min = 500 VM-min/h → **~8.3 CPU VMs**.
- Report generation can only go as fast as the GPU: 40 × 4 min = 160 VM-min/h → **~2.7 CPU VMs**.
- Peak CPU need is about **11 VMs**, inside the 14-CPU-worker budget, if placement is balanced.

### 1.3 VM budget (hard cap: 20 VMs, ≤ 4 GPU)

| Pool | AWS | GCP | Total |
|---|---|---|---|
| K8s system node (on-demand) | 1 | 1 | 2 |
| GPU workers (min–max) | 0–2 | 0–2 | ≤ 4 |
| CPU workers (min–max) | 0–7 | 0–7 | ≤ 14 |
| **Static per-cloud cap** | **10** | **10** | **20** |

- **Assumption:** the 20-VM limit covers VMs we provision (system + worker nodes). Managed
  services (RDS/Cloud SQL, EKS/GKE masters, Fargate/Cloud Run, object storage) are outside it.
  KEDA, CoreDNS and node agents need somewhere to run, hence 1 system node per cloud (v2).
- The cap must also hold during upgrades: surge disabled (EKS `MINIMAL` updates, GKE zero-surge).
- **Per-cloud static cap** instead of a dynamic global cap. If one cloud is unreachable we
  cannot prove its VMs are gone, so we must assume they still count against
  the limit. The surviving cloud therefore keeps its own 10 and does **not** borrow. In degraded mode the GPU
  rate is 20 jobs/h. Quota is shifted only after the other cloud's VMs are *confirmed* terminated.

### 1.4 Cold start

- GPU VM boot is about 2–3 min. CUDA + model image (~6–8 GB) pull is avoided by
  a pre-baked node image / lazy image streaming (GKE Image Streaming, SOCI on
  AWS). Model load is under 1 min. **Planning cold start is about 5 min.** That is small next to burst
  queueing, which is why scale-to-zero is acceptable.

---

## 2. Network

| Path | Assumption | Consequence |
|---|---|---|
| Partner → object storage | aggregate burst 100 × 3 GB / 1 h = 300 GB/h ≈ **670 Mbps**; per partner ≥ 100 Mbps → 3 GB in about 4 min | **Direct multipart upload with presigned URLs** (resumable, per-part checksums). The API never carries file bytes |
| VM ↔ object storage (same cloud) | ≥ 5 Gbps | 3 GB in about 5–10 s, negligible next to compute |
| AWS ↔ GCP | GCP HA VPN ↔ 2 AWS Site-to-Site VPN connections = 4 IPsec tunnels × ~1.25 Gbps (AWS per-tunnel limit). A single flow cannot exceed one tunnel, so plan on **1 Gbps effective** (to be measured); same-metro RTT < 5 ms | Enough for metadata, reports and occasional spill. A dedicated interconnect is not justified at < 10 TB/month |
| Metadata DB replication | KB/s | negligible |

---

## 3. Pricing (indicative only, to justify trade-offs)

| Item | Planning figure |
|---|---|
| Internet / inter-cloud egress | **~$0.12/GB** (published Sydney rates are about $0.09–0.19/GB depending on provider/tier) |
| Ingress | free |
| Object storage standard / infrequent / archive | ~$0.025 / ~$0.0125 / ~$0.002–0.005 per GB-month |
| GPU VM (A10G `g5.xlarge` / L4 `g2-standard-4`) | ~$1.0–1.5/h on-demand, planning **$1.2/h** |
| CPU VM (4 vCPU / 16 GB) | ~$0.2/h on-demand, ~60–70% less on spot |

### 3.1 Egress is the dominant multi-cloud cost

| Strategy | Monthly cross-cloud bytes | Cost |
|---|---|---|
| Naive: every job's raw input crosses clouds | 300 GB/day × 30 = 9 TB | **≈ $1,080/mo** |
| **Data-local** (stages run where the input lives; only reports + metadata cross) | ~30 GB | **≈ $4/mo** |
| Spill one job's raw input | 3 GB | $0.36/job |
| **Spill one job after prep** (ship prepared images, ~150 MB) | 0.15 GB | **≈ $0.02/job** |

→ **Placement rule:** prep always runs in the input's home cloud. The GPU stage
may spill to the other cloud when the home GPU queue wait exceeds a threshold, and then
it ships the *prepared* output instead of the raw batch, which is about 20× cheaper. Report
inputs are small, so report generation can run in either cloud.

### 3.2 GPU: scale-to-demand vs always-on

- Always-on: 4 × 720 h × $1.2 ≈ **$3,450/mo**
- Scale-to-demand: 10 GPU-h/day × 30 × $1.2 ≈ $360, plus 1 warm GPU 12 h/day
  ≈ $430, for a total of **≈ $800/mo**. The cost is up to about 5 min of extra latency on the first job.

### 3.3 Spot vs on-demand

- CPU stages run on **spot/preemptible** instances. A preemption is just another VM failure, and the
  lease mechanism already handles VM failures.
- GPU runs **on-demand**. Spot GPU capacity in Sydney is scarce and we have only 4 GPU slots, so
  losing one costs too much.

---

## 4. Storage tiers and retention

| Data | Store | Tier / lifecycle | Durability measures |
|---|---|---|---|
| Raw inputs | per-tenant bucket in home cloud | Standard; delete 30 days after job reaches terminal state `[CONFIRM: assume partners keep originals]` | 11 nines object durability; versioning |
| Intermediates | same bucket, `jobs/{id}/stage{n}/attempt-{k}/` | delete 7 days after job is terminal; orphan attempts are GC'd after 24 h | immutable once written |
| Reports | per-tenant bucket, **replicated to both clouds** | Standard → Infrequent at 30 days; keep 1 year `[CONFIRM]` | versioning + object lock (WORM); cross-cloud copy |
| Job metadata | PostgreSQL (AWS RDS primary, GCP Cloud SQL async replica) | PITR 14 days | Multi-AZ + cross-cloud replica + daily snapshot |
| Receipts (acceptance / publication / completion) | small JSON records mirrored to **both** clouds before the state becomes visible | kept with the job | let a promoted replica reconcile lost transitions without re-publishing differently |
| Logs / metrics | each cloud's native stack + central copy | 30 d hot, 1 y archive | |

Steady-state input storage: 9 TB × $0.025 ≈ **$225/mo**. Everything else is negligible.

---

## 5. Supplied programs and model

- Each stage is a CLI program: reads an input directory, writes an output
  directory, and exits 0 on success and non-zero on failure. It is treated as a black box, with
  no progress output → **stall detection uses heartbeat + resource/output progress + per-stage timeout**.
- Re-running a stage on the same input is safe because every attempt writes to a fresh directory.
  GPU inference may not be bit-for-bit deterministic. That is acceptable, because exactly one attempt is *published*.
- The model fits on one GPU with ≤ 24 GB VRAM. Weights (~1 GB assumed) are stored in our own
  buckets in both clouds and identified by SHA-256. Nothing downloads from public hubs at runtime.
- The programs and model may be redistributed inside our private registries (licence assumption).

---

## 6. Third-party dependencies

| Dependency | Choice | If unavailable |
|---|---|---|
| Partner identity | external OIDC IdP (e.g. the university's Entra ID / Auth0) `[CONFIRM]` | new logins fail; existing tokens (≤ 1 h) keep working; running jobs are unaffected |
| DNS / global entry | **provider-neutral DNS with health checks** (e.g. Cloudflare), so failover does not depend on the cloud that just failed | static fallback hostnames per cloud |
| Container registry | ECR + Artifact Registry, mirrored; images pinned **by digest** and signed (cosign) | each cloud pulls from its local registry |
| IaC / CI | Terraform (state in S3 + DynamoDB lock), GitHub Actions with OIDC federation to both clouds | deployments pause; runtime is unaffected |
| Control-plane failover witness | small serverless leadership lease outside both clouds (or the DNS provider's health checks) | failover becomes manual (operator promotes the replica) |
