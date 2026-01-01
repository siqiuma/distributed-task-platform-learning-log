# How We Observe the System (Logs, Metrics, Health)

This project runs a background **TaskWorker** that repeatedly:
1) finds eligible tasks (PENDING or FAILED and due),
2) claims exactly one task (race-safe),
3) processes it outside a DB transaction,
4) marks it SUCCEEDED or FAILED (with retry/backoff).

This document explains how we observe that worker in production.

---

## 1) Logs (decision logs)

### What “good” looks like
Logs should answer:
- **Was a task claimed?**
- **Was it processed?**
- **Did it succeed or fail?**
- **If it failed, was retry scheduled and when?**
- **Did we hit concurrency conflicts?**

### Key log events emitted by `TaskWorker.runOnce()`
- `task_skipped ... reason=already_claimed_or_not_due`
  - Worker saw a task but the DB claim did not update a row → another worker got it, or task was no longer due.
- `task_claimed ... status=PROCESSING attempt=... maxAttempts=...`
  - Claim succeeded and attemptCount was incremented in DB.
- `task_processing_started ... payloadLen=...`
  - “Work” begins outside the transaction.
- `task_succeeded ... durationMs=...`
  - Work completed and task state updated to SUCCEEDED.
- `task_processing_failed_expected ...`
  - Expected/app-level failure (example: simulated failure).
- `task_processing_failed_unexpected ...`
  - Unexpected crash/infra bug; includes stacktrace at ERROR.
- `task_retry_state_updated ... nextRunAt=...`
  - Task was marked FAILED and retry scheduling state was written.
- `task_conflict_optimistic_lock ...`
  - Concurrency conflict (rare), worker skips safely.
- `task_markFailed_failed ...`
  - Mark-failed path itself failed (should be rare, investigate).

### Log level rules
- **INFO**: normal lifecycle events (claimed/started/succeeded/retry-updated).
- **WARN**: expected failures, optimistic-lock conflicts, interrupts.
- **ERROR**: truly unexpected failures (bugs, infra issues) or failure to update retry state.

---

## 2) Metrics (Actuator / Prometheus)

### Endpoints
Assuming Spring Boot Actuator + Prometheus registry are enabled:

- List all metric names:
  - `GET /actuator/metrics`

- Inspect a single metric:
  - `GET /actuator/metrics/dtp_tasks_processed_total`

- Prometheus scrape output:
  - `GET /actuator/prometheus`

### Worker metrics (Prometheus-friendly)
These are registered in `TaskWorker`:

- `dtp_tasks_processed_total` (counter)
  - Incremented once per task that `runOnce()` attempts to handle (success or failure).
- `dtp_tasks_succeeded_total` (counter)
  - Incremented when task is marked SUCCEEDED.
- `dtp_tasks_failed_total` (counter)
  - Incremented when task is marked FAILED (retry scheduled or terminal failure).
- `dtp_tasks_claim_conflicts_total` (counter)
  - Incremented when `claimAndGetSnapshot()` returns null (not claimed).
- `dtp_tasks_mark_failed_errors_total` (counter)
  - Incremented when `markFailed` path throws an unexpected exception.
- `dtp_task_processing_duration_seconds` (timer → exported as histogram)
  - Measures time spent processing outside the transaction.

### Where the “Histogram” is
In `/actuator/prometheus`, histograms appear as multiple time series:

- Buckets:
  - `dtp_task_processing_duration_seconds_bucket{le="0.5", ...} ...`
- Count:
  - `dtp_task_processing_duration_seconds_count ...`
- Sum:
  - `dtp_task_processing_duration_seconds_sum ...`
- (Optional) Max:
  - `dtp_task_processing_duration_seconds_max ...`

If your output currently shows `_bucket` lines but `*_count` is `0`, it means **no timing samples were recorded** yet (or the timer stop logic didn’t execute as expected). Once tasks are processed and the timer is stopped exactly once per task, the bucket counts should increase.

---

## 3) Health / Readiness (Actuator)

### Meaning
We use readiness to answer:
> “Is this instance able to do useful work right now?”

For the worker, “ready” usually means:
- DB is reachable (worker can query & claim tasks)
- Worker is enabled (not disabled via config)

### What “worker” means
“Worker” is the background processing loop: finding/claiming/updating tasks.
It’s not checking “TaskWorker.java is alive” directly; instead it checks the *dependencies required for the worker to operate* (primarily DB connectivity and configuration).

### Recommended readiness design
A worker readiness indicator should validate:
- `task.worker.enabled == true`
- a lightweight DB check succeeds (example: `SELECT 1` / repository ping)

If DB is down, readiness should be **DOWN** because the worker cannot claim or update tasks.

---

## 4) Prometheus + Grafana (local setup)

### Where to place `prometheus.yml`
Create it at the repo root (same level as `pom.xml`), for example:
- `./observability/prometheus.yml`  (recommended)
or
- `./prometheus.yml`

Example `observability/prometheus.yml`:
```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: "dtp"
    metrics_path: "/actuator/prometheus"
    static_configs:
      - targets: ["host.docker.internal:8080"]
