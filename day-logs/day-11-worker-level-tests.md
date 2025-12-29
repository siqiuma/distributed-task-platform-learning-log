### Lesson 10 — Worker Correctness Tests

#### Goal
- Prove that the task worker behaves correctly under real retry conditions
- Validate **claiming, success, retry, delay, and terminal failure** paths end-to-end
- Ensure behavior matches domain + repository rules already tested in Lessons 7.1 and 7.2

---

### Test Setup
- Used `@SpringBootTest` with `test` profile
- Backed by **Postgres Testcontainers** (real DB behavior)
- No mocking of `TaskRepository` or `TaskWorker`
- Assertions verify **database state**, not logs or mocks

---

### Tests Implemented

#### 1. Worker claims and succeeds exactly once
- Create a `PENDING` task with normal payload
- Run `worker.runOnce()`
- Assert:
  - Task moves to `SUCCEEDED`
  - `attemptCount` increments to `1`
  - Task is not processed again on subsequent runs

---

#### 2. Worker marks task as FAILED and schedules retry
- Create a task with payload containing `"fail"` (simulated failure)
- Run `worker.runOnce()`
- Assert:
  - Task moves to `FAILED`
  - `attemptCount = 1`
  - `lastError` populated
  - `nextRunAt` scheduled in the future (backoff applied)

---

#### 3. Worker does NOT process FAILED task when retry is not due
- Manually create a `FAILED` task with `nextRunAt` in the future
- Run `worker.runOnce()`
- Assert:
  - Task is skipped
  - No state change occurs

---

#### 4. Worker processes FAILED task when retry is due
- Create a `FAILED` task with `nextRunAt <= now`
- Run `worker.runOnce()`
- Assert:
  - Task is claimed again
  - Attempt count increments
  - Task transitions according to success/failure logic

---

#### 5. Worker stops retrying after max attempts (terminal failure)
- Create a failing task
- Force `max_attempts = 1` via `JdbcTemplate`
- Run `worker.runOnce()`
- Assert:
  - Task transitions to `DEAD`
  - `attemptCount = 1`
  - `nextRunAt = null`
- Run `worker.runOnce()` again
- Assert:
  - Task remains `DEAD`
  - No further processing occurs

---

### Key Learnings
- Worker correctness depends on **repository row-count semantics**, not locks
- Retry behavior is enforced by **state + timestamps**, not sleep-based timing
- `DEAD` is a true terminal state and safely excluded from eligibility
- Tests validate **real transactional boundaries**, not mocked behavior

---

### Outcome
- Worker logic is now **provably correct**
- Retry system is safe, bounded, and deterministic
- Ready to move into **Observability (logs, metrics, health checks)**

