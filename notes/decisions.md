# Design Decision Log (final)

Each entry records the final choice, the main alternative, and why. The report's §10 table is the
condensed version. "History" notes how the design changed between drafts.

---

## D1. Clouds and regions — AWS + GCP, both in Sydney
- **Why:** both offer managed Kubernetes with GPU node pools, HA VPN to each other and workload
  identity. Sydney is the only Australian region where both offer the same GPU (T4); GCP Melbourne
  has no GPUs. Same metro keeps inter-cloud RTT low enough for synchronous replication, and data
  stays in Australia (camera traps also photograph people).
- **Alternative:** Sydney + Melbourne (more geographic independence, but no GCP GPUs and higher RTT).

## D2. Multi-cloud strategy — active-active compute, quorum-governed state
- Both clouds run job services and workers daily; one PostgreSQL leader at a time, elected by a
  three-voter etcd quorum (AWS, GCP, third-site witness).
- **Alternative:** async replica with operator-run promotion (no automatic write failover); multi-master
  DB (needs ≥ 3 data regions, too heavy for 100 jobs/day).

## D3. Job store and queue — PostgreSQL (`SELECT … FOR UPDATE SKIP LOCKED`)
- One transaction covers state, queue and leases, so no dual-write. Load is < 1 write/s.
- **Alternative:** SQS / Pub/Sub plus a database.

## D4. Work assignment — pull-based fenced leases
- Runners request leases with their capabilities; the epoch increments only on a claim; 90 s lease,
  renewed every 30 s. Discovery and load balancing come for free.
- **Alternative:** push behind an L4 load balancer. SmartPark (A1) showed why not: L4 balances per
  connection, and after scale-out the new pod got 3 m CPU vs 994 m on the old one.

## D5. Orchestration — EKS + GKE, KEDA ScaledJobs, single-use pods
- One pod per attempt (fresh filesystem per tenant job), KEDA `accurate` strategy, GPU taints,
  required anti-affinity (one attempt per worker VM).
- **Alternative:** CPU-based HPA (A1); CPU cannot see a GPU backlog.

## D6. Result commit protocol (failure scenario)
- Create-only attempt prefixes (S3 `If-None-Match`, GCS `ifGenerationMatch=0`), manifest written
  last; conditional `UPDATE` checking job, stage, attempt, owner, epoch, `RUNNING` and
  `lease_expires_at > clock_timestamp()`; next stage queued in the same transaction.
- Storage credentials are not the fence (STS sessions last ≥ 15 min).
- **History:** v1 incremented the epoch in both reaper and claim and assumed credentials expired with
  the lease; both corrected.

## D7. Orphan adoption — rejected
- Adopting a verified orphan saves ≈ 6 GPU-min but adds a second publication path. Rerun instead.

## D8. Metadata HA — Patroni, synchronous standby, 3-voter etcd
- `synchronous_commit=on`; only members in `/sync` can be promoted; a member leaves `/sync` (quorum
  write) before the leader stops waiting for it; leader key TTL 30 s + watchdog; clients never cancel
  `COMMIT`.
- Result: automatic failover ≈ 30–60 s, RPO = 0 for acknowledged commits.
- **Cost:** self-managed database, a few ms per commit, third-site witness dependency.
- **History:** v2 used receipts + a backlog flag (holes: stale backlog report, timeouts are not
  fencing, admission conflict after takeover); a gated manual runbook was safe but had no automatic
  write failover. Quorum + synchronous replication fixes both.

## D9. Tenant isolation
- Tenant from the token, ownership checks, PostgreSQL RLS (`SET LOCAL`, non-owner roles without
  `BYPASSRLS`), one bucket and KMS key per tenant per cloud, per-lease scoped credentials,
  short-lived presigned URLs.

## D10. Repeated failures
- Transient program failures: ≤ 2 retries with backoff + jitter (OOM → larger pool). Deterministic
  input errors: not retried. Infrastructure failures: separate budget (5 attempts or 24 h), node
  quarantine after cross-job failures. Budgets are initial defaults, tuned by fault injection.

## D11. Capacity
- **VM split 10 + 10** (2 GPU each) rather than 20 in one cloud: two-provider compute, warm capacity
  survives a cloud loss, only 2 T4s needed per region, data stays local.
- **After losing a cloud:** keep 10 until the other cloud's VMs are confirmed terminated, then CI
  raises limits up to 20 VMs / 4 GPUs.
- **Job service ≥ 2 tasks:** a single restart could outlast the 90 s lease and void in-flight GPU work;
  ≈ $40–45/month.

## D12. Deployment and provisioning
- Terraform for both clouds, run from CI (plan on PR, apply after review); A1 used a one-off
  `gcloud container clusters create`.
- Plain YAML + `kubectl apply` from CI with signed image digests, scheduled `kubectl diff` for drift;
  Helm + Argo CD rejected as extra controllers for two clusters.
- Model weights fetched by an init container, pinned by version and SHA-256 (A1 pulled the latest
  object without a checksum).
- Provider-managed front ends (ALB, Cloud Run) instead of a self-configured GCE Ingress (A1's never
  became healthy).

## D13. Caching
- No application result cache: batches are unique, status reads are cheap, and a second store would
  break the single source of truth. Idempotency keys and stored reports cover repeats; only
  infrastructure caches remain (model, image layers, IdP JWKS). A1 used a per-pod in-memory cache
  (TTL 60 s, key = user uuid + number of car parks) because it served repeated real-time queries.
