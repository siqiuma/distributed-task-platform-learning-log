# Day 13 Lesson 12 Worker Metrics

This document describes the Prometheus metrics emitted by the Distributed Task Platform worker.
These metrics are designed for operational visibility, alerting, and capacity planning.

---

## Counters

### `dtp_tasks_processed_total`
**Type:** Counter  
**Description:**  
Total number of tasks the worker attempted to process (successful or failed).

**Use cases:**
- Verify worker activity
- Detect stalled workers (no increase over time)

---

### `dtp_tasks_succeeded_total`
**Type:** Counter  
**Description:**  
Total number of tasks that completed successfully.

**Use cases:**
- Success rate calculation
- Throughput monitoring

---

### `dtp_tasks_failed_total`
**Type:** Counter  
**Description:**  
Total number of task executions that resulted in a failure and triggered retry or terminal failure logic.

**Use cases:**
- Failure rate monitoring
- Alerting on abnormal error patterns

---

### `dtp_tasks_claim_conflicts_total`
**Type:** Counter  
**Description:**  
Number of times a task could not be claimed due to concurrent processing or optimistic locking.

**Use cases:**
- Detect worker contention
- Identify scaling or locking issues

---

### `dtp_tasks_mark_failed_errors_total`
**Type:** Counter  
**Description:**  
Number of unexpected errors that occurred while attempting to mark a task as failed.

**Use cases:**
- Detect infrastructure or persistence-layer failures
- Alert on retry-state corruption risks

---

## Timers / Histograms

### `dtp_task_processing_duration_seconds`
**Type:** Histogram  
**Description:**  
Time spent processing a task outside of database transactions.

This metric measures only the business logic execution time, not database or claim overhead.

**Use cases:**
- p95 / p99 latency monitoring
- Performance regression detection
- Capacity planning

---

## Derived Metrics (Prometheus Queries)

### Task Success Rate
```promql
rate(dtp_tasks_succeeded_total[5m])
/
rate(dtp_tasks_processed_total[5m])
