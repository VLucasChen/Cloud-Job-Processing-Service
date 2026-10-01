# Design Decision Log

One entry per key choice: options, recommendation, trade-off. The
**Decision** line is yours to fill in. Write *why* in your own words, because
these become the "justify key choices and trade-offs" sentences in the report.

Status legend: `PROPOSED` (recommended, awaiting decision) ·
`ACCEPTED` · `REJECTED`.

---

## D1. Which two clouds — `PROPOSED`

- **Options:** AWS + GCP · AWS + Azure · GCP + Azure
- **Recommendation:** AWS + GCP. Both have managed Kubernetes with GPU
  node pools (EKS / GKE), HA VPN to each other, workload-identity federation in both
  directions, and good GPU supply in Sydney.
- **Trade-off:** Azure would fit better if the university already uses Entra ID.
  Partner login is federated OIDC anyway, so this barely matters.
- **Decision:**

## D2. Multi-cloud strategy — `PROPOSED`

- **Options:**
  1. Active-passive for everything (GCP idle until failover)
  2. **Compute active-active, control plane active-passive**
  3. Fully active-active (multi-master DB such as CockroachDB/Spanner)
- **Recommendation:** (2). Both clouds' workers run jobs every day, so failover
  capacity is always warm and tested. The metadata DB has one writer (AWS) and an
  async replica in GCP that gets promoted on failure.
- **Trade-off:** (3) needs ≥ 3 regions for quorum and adds a third-party
  dependency. That is overkill at 100 jobs/day. The cost of (2) is an RPO of a few seconds for
  metadata, which is covered by the dual-written acceptance journal (D8) and idempotent
  worker re-reporting.
- **Decision:**

## D3. Queue technology — `PROPOSED`

- **Options:** SQS / Pub/Sub · RabbitMQ/Kafka · **PostgreSQL as the queue**
  (`SELECT … FOR UPDATE SKIP LOCKED`)
- **Recommendation:** Postgres. Job state, leases and the queue sit in one
  transactional store, so there is no dual-write between a queue and a DB (the classic source of
  lost or duplicated jobs). Load is tiny (< 1 operation/s).
- **Trade-off:** it is not a "cloud-native" queue, and the DB becomes the critical
  component. Accepted because the DB is already replicated (D2) and SQS would not
  remove that dependency anyway (state still lives in the DB).
- **Decision:**

## D4. Work assignment: pull vs push — `PROPOSED`

- **Recommendation:** **pull with leases.** An agent on each worker node polls
  the coordinator API with its capabilities (`cloud`, `gpu`, `mem`, `free slots`)
  and receives a lease `(job, stage, attempt, epoch, ttl)`. Worker discovery
  falls out of this for free: a worker exists only while it holds or
  asks for leases.
- **Placement score:** must match resources (GPU stage → GPU pool) → prefer data-local
  cloud → least-loaded node. GPU spill to the other cloud happens only when the home
  queue's estimated wait is greater than transfer time + a threshold (assumptions §3.1).
- **Trade-off:** polling adds up to a few seconds of latency, which is irrelevant against
  5–30-minute stages. Push would need a registry of workers plus failure detection, which
  pull already gives us.
- **Decision:**

## D5. Orchestration — `PROPOSED`

- **Options:** plain VMs + Docker + systemd agent · **managed K8s per cloud
  (EKS + GKE)** · Nomad multi-region
- **Recommendation:** EKS + GKE. Use node pools with taints (GPU pool only runs
  the identify stage), cluster autoscaler with hard `max` per pool (enforces the VM
  cap), one Kubernetes Job per stage attempt, NetworkPolicies, and workload identity. Our
  coordinator is the *cross-cloud* scheduler, and K8s only runs pods within one cloud.
- **Trade-off:** running two clusters is operationally heavy for a small team.
  Plain VMs are simpler but would need hand-built isolation, scaling and health checks.
- **Decision:**

## D6. Result commit protocol (core of the failure scenario) — `PROPOSED`

1. Every attempt writes only to its own immutable prefix
   `jobs/{id}/stage{n}/attempt-{k}/`, with `_MANIFEST.json` (file list +
   SHA-256) written **last**.
2. Every lease carries a monotonically increasing **fencing epoch**. When a lease expires,
   the coordinator increments the epoch before reassigning the stage.
3. Publishing is a conditional update:
   `UPDATE stage SET status='SUCCEEDED', output=attempt-k WHERE job=… AND stage=n AND epoch=:e AND status='RUNNING'`.
   Only the holder of the current epoch can succeed. Downstream stages read
   **only** the published pointer and never list a prefix.
4. A stale worker that comes back gets `409 stale epoch`. It stops and its prefix
   is garbage-collected.
- **Decision:**

## D7. Adopt a completed orphan attempt instead of recomputing? — `PROPOSED (optional)`

- When a lease expires, the coordinator first checks whether the expired attempt has a
  complete `_MANIFEST.json` whose checksums verify. If so, it publishes *that* attempt (and bumps the epoch
  so the original VM's late commit is still rejected) and skips the rerun.
- **Gain:** saves about 6 GPU-min per incident, and GPU is the bottleneck.
- **Cost:** extra code path. It is only safe because objects are immutable and the
  manifest is written last.
- **Decision:**

## D8. "Accepted jobs are never silently lost" — `PROPOSED`

- A job is ACKed (HTTP 202 + job ID) only after: (a) the upload is complete and checksum-verified,
  (b) the DB row is committed, and (c) a small acceptance-journal record is written to **both**
  clouds' object storage. If the AWS DB is lost before the row replicates, the
  promoted GCP DB rebuilds missing jobs from the journal.
- Workers buffer stage-completion reports locally and retry with idempotency keys,
  so completions survive control-plane failover.
- **Decision:**

## D9. Tenant isolation — `PROPOSED`

- **Options:** shared bucket + prefix ACLs · **bucket per tenant per cloud +
  KMS key per tenant**
- **Recommendation:** bucket + key per tenant. With few partners (assume ≤ 20) it is
  cheap. IAM policy is simple. Deleting a tenant's key crypto-shreds its data at offboarding.
  The DB uses `tenant_id` on every row + PostgreSQL row-level security, keyed
  on the authenticated token's tenant claim.
- **Worker credentials:** short-lived, scoped down to one job's prefixes
  (read input prefix, write own attempt prefix). They are issued per lease through workload identity.
- **Decision:**

## D10. Repeated-failure policy — `PROPOSED`

- Infrastructure failure (lease expiry, VM lost, preemption) → retry immediately on a
  different node. It does **not** count against the job's attempt budget, but it counts
  against the *node's* health score (node cordoned after 2 failures from different jobs within 1 h).
- Program failure (non-zero exit, OOM) → retry ≤ 2 more times with backoff, on a
  larger VM if the failure was OOM. After that → `FAILED_NEEDS_REVIEW` (poison job). Input is
  retained, the partner and operator are notified, and the queue keeps going.
- **Decision:**

## D11. Report tooling — `PROPOSED`

- LaTeX (`report/main.tex`, Overleaf-compatible) for tight control of the 5-page
  limit. Diagram as vector PDF/SVG (draw.io, or the Python `diagrams` package for
  real cloud icons).
- **Decision:**
