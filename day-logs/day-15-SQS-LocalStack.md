Day 15 SQS (LocalStack + Spring Boot + Postgres) 
--------------------------------------------------------------

### Goal

Prove an end-to-end task lifecycle using **Postgres as source of truth**, **SQS as transport**, and a **Spring Boot worker** that processes tasks and records outcomes reliably.

### What I built / validated

-   **Local SQS (LocalStack)** integrated with Spring Boot via `SqsClient` configured using endpoint override + static credentials.

-   **Queue URL resolution at startup** using `GetQueueUrl(queueName)`; queue name is `dtp-task-queue`.

-   **SQS worker loop**

    -   long-polling `ReceiveMessage(waitTimeSeconds=20, max=5, visibilityTimeout=30)`

    -   per-message handling with idempotency and DB lifecycle completion

-   **DB lifecycle implemented in JDBC**

    -   `claimEnqueuedTask(taskId, workerId)`

        -   atomically transitions `ENQUEUED → PROCESSING`

        -   increments `attempt_count`

        -   checks `scheduled_for <= now()` and `attempt_count < max_attempts`

    -   `markSucceeded(taskId, workerId)`

        -   transitions `PROCESSING → SUCCEEDED`

        -   writes `completed_at`

    -   `markFailedAndReschedule(taskId, workerId, error, backoffSeconds)`

        -   if exhausted => `DEAD` and set `completed_at`

        -   else => reschedule by setting `scheduled_for = now() + backoff`

-   **Correct message deletion semantics**

    -   Delete SQS message **only after DB update succeeds** (success or failure path).

    -   If DB update fails (wrong worker/status, etc.), **do not delete**, allowing redelivery.

-   **Scheduling model confirmed**

    -   Keep scheduling via `scheduled_for` (DB is the scheduler).

    -   SQS delivery is "wake-up / work notification", but DB controls eligibility.

### Metrics / Observability

-   Prometheus `/actuator/prometheus` working end-to-end.

-   Confirmed counters + lag tracking update correctly:

    -   `dtp_sqs_messages_received_total`

    -   `dtp_sqs_messages_deleted_total`

    -   `dtp_tasks_processed_total`

    -   `dtp_tasks_succeeded_total`

    -   `dtp_task_schedule_lag_seconds_*` histogram reflects real lag

### Key bugs I hit (and fixes)

1.  **Local Postgres "http://localhost:5432" confusion**

    -   Postgres isn't HTTP; connect via `psql` or a DB client (JDBC).

2.  **SQS "queue does not exist"**

    -   Health said SQS was up, but queue not created / wrong endpoint/region.

    -   Fix: ensure queue exists and app points to LocalStack endpoint + correct region.

3.  **State machine / test failures**

    -   Domain tests expected transitions that no longer matched (e.g., enqueuing after failure).

    -   Fix: updated allowed transitions + aligned tests.

4.  **Integration test startup ordering**

    -   Worker bean resolves queueUrl at startup, so queue must exist before context loads.

    -   Fix: `@BeforeAll` createQueue + `dtp.sqs.worker.autostart=false` to avoid background thread during IT.

### Tests completed

-   ✅ Unit tests (Mockito) for worker behavior (`TaskWorkerTest` passed).

-   ✅ Repository tests for claim/markSucceeded/markFailedAndReschedule.

-   ✅ **SQS end-to-end integration test passed**

    -   Due task in DB → enqueuer sends SQS message → worker consumes → DB marked `SUCCEEDED`.

### Current design decisions (intentional)

-   **DB is source of truth** for retries and scheduling.

-   SQS is transport; **don't rely on SQS redelivery for retry logic**.

-   Message deletion happens only after DB records final outcome for that attempt.

### Next high-value milestone (planned)

-   Add **DLQ** semantics (route poison / DEAD tasks for inspection)

-   Add crash recovery for stuck PROCESSING tasks (reaper job)

-   Add multi-worker contention test (prove horizontal scaling)
