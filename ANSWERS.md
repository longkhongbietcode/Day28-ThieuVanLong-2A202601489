# Answers — Day 28 Track 2 (Lab 28 Platform)

## 1. Trade-offs

- **Delta MERGE dedupe done in Python, not Spark SQL.** `dedupe_latest`
  (`src/lab28_platform/integration_tasks.py`) reduces the batch to one row per
  `idempotency_key` in a single pass over the iterable before the MERGE runs,
  instead of deduping inside the Spark SQL statement. This keeps the merge
  key logic testable without a Spark session (`starter-tests` run in
  milliseconds), at the cost of doing the reduction on the driver instead of
  distributed — acceptable at this event volume, would need to move into the
  Spark job itself before ingest volume grows past what a single driver can
  hold in memory.
- **`readiness_status` treats "not_ready" and "degraded" as mutually
  exclusive severities, evaluated in fixed priority order** rather than
  computing a weighted score. This matches how the gateway consumes `/ready`
  (binary remove-from-rotation decision) but throws away information a more
  granular SLO dashboard might want (e.g. *how many* non-mandatory probes are
  failing, not just whether any are).
- **Kafka header propagation omits `traceparent` entirely when absent**
  instead of sending an empty string. This is correct per the W3C trace
  context spec (an empty header is an invalid trace context, not "no trace"),
  but it means any consumer must handle the *absence* of the key, not assume
  it is always present with a possibly-empty value.
- **Local vLLM was not stood up.** The platform is wired to run inference
  against a real vLLM server (Kaggle T4 or a shared GPU box per
  `KAGGLE_GPU_EXTENSION.md`), and this environment has no GPU. Rather than
  fake an OpenAI-compatible stub (explicitly disallowed by the rubric), the
  `/api/v1/ask` path and the serving-side trace spans
  (`lab28.api.ask`, `lab28.qdrant.query`, `lab28.feast.get_online_features`,
  `lab28.mlflow.resolve_release`, `lab28.vllm.chat_completion`) are left
  genuinely `UNVERIFIED` in this run's evidence.

## 2. Production gaps

- **IP07 (vLLM) is not_ready in this environment.** `evidence/ip07-vllm-identity.json`
  records `"reachable": false"` — no GPU endpoint was available. To close
  this gap: point `VLLM_BASE_URL` at a real Kaggle T4/shared GPU vLLM
  instance per `KAGGLE_GPU_EXTENSION.md`, then re-run `lab28 evidence` and
  the J1/J5 integration tests so `/api/v1/ask` and the five missing spans in
  `evidence/ip10-trace.json` get produced.
- **K8s/GitOps is validated statically, not exercised live.** `scripts/validate_manifests.py`
  passes (manifests are structurally correct and internally consistent with
  `gitops/application.yaml`), but there is no running kind/minikube cluster
  or Argo CD instance in this environment to actually demonstrate drift
  detection, self-heal, or a desired-state rollback (`runbooks/gitops-rollback.md`
  steps 3–5). This is a real gap for the "production readiness" rubric line
  and needs a cluster to close, not more local docker-compose work.
- **No alert has fired in anger.** Prometheus rules are provisioned and
  `evidence/ip09-prometheus-targets.json` shows every job up, but the demo
  runbook's incident step (predict → inject → observe → recover → prove no
  data loss) was not run against a live alert firing end-to-end in this
  session — only the automated `test_j4_degraded_recovery` failure-injection
  path was exercised via the integration suite.
- **Rate limiting is aggressive for a cold seed.** `lab28 seed --via-gateway`
  consistently gets 5 of its feedback requests rejected with
  `local_rate_limited` (429) on a fresh run — expected gateway behavior
  (IP08's local rate-limit policy), but it means a real onboarding/demo
  script needs to either seed slower or treat a partial 429 batch as
  success, not retry blindly.
- **Windows console encoding.** `mlflow` writes emoji to stdout on run
  completion, which raises `UnicodeEncodeError` under the default Windows
  `cp1252` code page unless `PYTHONUTF8=1`/`PYTHONIOENCODING=utf-8` is set.
  Worth adding to the README's Windows setup notes so `lab28 release` doesn't
  look like a real failure on a fresh machine.

## 3. What was verified in this session

- `starter-tests` + `tests`: 87/87 passing after implementing the four
  functions in `integration_tasks.py` (event headers, replay-safe dedupe,
  Feast request, readiness severity).
- `ruff check .`, `scripts/verify_matrix.py`, `scripts/check_portability.py`,
  `scripts/validate_manifests.py`: all clean.
- `docker compose --env-file ports.template [--profile full] config --quiet`:
  both profiles valid.
- Default profile (`local-standard`) brought up and confirmed healthy via
  `lab28 topics`, `index`, `release`, `seed --via-gateway`, `inspect`, `ready`.
- Full profile (adds Airflow + Spark Connect) brought up; `IT-J1-golden-path`
  and `IT-J2-idempotent-replay` pass individually, then the whole
  `integration-tests -m "not gpu and not langsmith"` suite: **56 passed, 16
  deselected** (gpu/langsmith-gated).
- All 10 IP evidence files present in `evidence/` (IP07 correctly recorded as
  `not_ready`/unreachable rather than faked).
- Load profile: `load-tests/run_profile.py --requests 200 --workers 8` →
  200/200 status 200, latency P50 **766.6 ms**, P95 **1042.4 ms**, P99
  **2681.0 ms**. The P50→P99 spread (~3.5x) points at a small number of
  slow-tail requests rather than a systemic bottleneck; worth re-running with
  request-level tracing enabled to confirm which stage (embedding, Qdrant
  search, or MLflow artifact resolution) owns the tail before treating it as
  a real SLO risk.

## 4. Contribution

Individual submission — all four boundaries, verification gates, local
platform bring-up (default + full profile), evidence collection, and load
test were completed by one person, using the "Phân vai" table in `plan.md`
§8 as a checklist to cover every IP rather than to split work across people.
