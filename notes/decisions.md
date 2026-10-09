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
- **Recommendation:** (2). Both clouds run the stateless job service and workers
  every day, so failover capacity is always warm and tested. The only singleton is the
  metadata DB writer (AWS); GCP's job service writes to it over the VPN, and GCP holds an
  async replica that is promoted on failure (gated by a third-party witness to prevent
  split-brain).
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
- **Final toolchain:** plain YAML manifests + `kubectl apply` from CI (digest-pinned),
  scheduled `kubectl diff` for drift; Helm + Argo CD rejected as extra controllers for 2 clusters.
- **Recommendation:** EKS + GKE. Use node pools with taints (GPU pool only runs
  the identify stage), cluster autoscaler with hard `max` per pool (enforces the VM
  cap), NetworkPolicies, and workload identity. KEDA **ScaledJobs** create one
  **single-use runner pod per queued lease**: the pod claims one lease, runs one attempt,
  exits. Every tenant job therefore gets a fresh filesystem. API + coordinator are one
  stateless "job service" (Fargate / Cloud Run). It is the *cross-cloud* scheduler, and K8s
  only runs pods within one cloud.
- **Trade-off:** running two clusters is operationally heavy for a small team.
  Plain VMs are simpler but would need hand-built isolation, scaling and health checks.
- **Decision:**

## D6. Result commit protocol (core of the failure scenario) — `PROPOSED`

1. Every attempt writes only to its own prefix `jobs/{id}/stage{n}/attempt-{uuid}/` with
   **create-only** writes (S3 `If-None-Match`, GCS `ifGenerationMatch=0`); `_MANIFEST.json`
   (object versions + SHA-256) is written **last**.
2. The epoch is incremented **only when a lease is claimed** (with a new attempt UUID).
   The reaper does not change it: it marks the expired attempt `ABANDONED` and requeues.
3. Publishing is a conditional update that checks job, stage, attempt UUID, owner, epoch,
   `status='RUNNING'`, not cancelled, and `lease_expires_at > now()` (DB clock). It sets
   `PUBLISHING`; once the publication receipt is mirrored to both clouds, a second
   transaction sets `SUCCEEDED` and queues the next stage. Downstream reads only the
   published manifest.
4. A stale worker that comes back gets `409 STALE_LEASE`; its prefix is GC'd after 24 h.
5. Storage credentials are **not** the fence: AWS STS sessions last ≥ 15 min, longer than
   the 90 s lease. They are scoped to the attempt prefix, which is never read unless published.
- **v2 change:** v1 incremented the epoch in both the reaper and the claim (inconsistent)
  and claimed credentials expire with the lease (wrong).
- **Decision:**

## D7. Adopt a completed orphan attempt instead of recomputing? — `REJECTED in v2`

- When a lease expires, the coordinator first checks whether the expired attempt has a
  complete `_MANIFEST.json` whose checksums verify. If so, it publishes *that* attempt (and bumps the epoch
  so the original VM's late commit is still rejected) and skips the rerun.
- **Gain:** saves about 6 GPU-min per incident, and GPU is the bottleneck.
- **Cost:** extra code path. It is only safe because objects are create-only and the
  manifest is written last.
- **v2:** rejected to keep a single publication path; mentioned as "considered" in §7.
- **Decision:**

## D8. Writer failover: gated promotion, no receipts — `ACCEPTED (final)`

- Acceptance = one durable commit on the Multi-AZ primary (no synchronous cross-cloud copy).
- Reports are copied and verified in both clouds before `COMPLETED`; a completion-catalog entry
  is mirrored to GCP afterwards for outage reads.
- Routing (DNS, runners) fails over automatically. **Writer** failover is a scripted,
  operator-approved runbook: pause → fence the old writer via the AWS control plane →
  verify the standby applied the final WAL position → promote once, bump generation.
  If fencing or completeness cannot be proven, stay read-only and wait.
- **Why final:** the v2 receipt/backlog protocol had real holes (a stale backlog=0 report;
  statement timeouts don't bound transactions or paused clients; post-takeover admission
  contradicted the dual-receipt rule). Patching it would add more protocol than can be
  explained and defended. Trade-off: no bounded write RTO in an ambiguous partition.

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
