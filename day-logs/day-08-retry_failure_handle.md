## Day 8 – Lesson 7 (Part 2): Retry Semantics, State Correctness, and Failure Handling

### What I worked on
- Continued implementing and validating retry logic for the task worker.
- Reviewed and corrected retry-related domain logic (`FAILED` vs `DEAD`, `nextRunAt`, `attemptCount`).
- Added and refined `GET /tasks/{id}` to clearly expose retry state and execution history.
- Walked through worker logic in detail to understand how failures are signaled and handled.
- Verified retry behavior through API calls and observed state transitions.
- Identified and fixed logical inconsistencies in retry eligibility and attempt counting.
- Asked detailed questions about Spring Data JPA mechanics (`@Param`, `:id`, `@Modifying`, `Pageable`).
- Deeply reviewed exponential backoff logic and atomic task claiming.
- Challenged assumptions in provided code and reasoned through corrections.

### What I learned
- **Retry logic is a state machine**, not just a loop:
  - `PENDING → PROCESSING → FAILED (retryable) → PROCESSING → … → DEAD`
- **Terminal failure must be represented explicitly**:
  - Introducing a `DEAD` status prevents infinite retries and simplifies eligibility queries.
- **`nextRunAt` is a “not-before” timestamp**:
  - `nextRunAt <= now` means the task is eligible to retry *now*.
- **Attempt counting semantics must be consistent**:
  - Incrementing attempts when claiming (starting work) avoids off-by-one errors.
- **Failure handling should be centralized**:
  - Workers throw exceptions to signal failure; domain logic decides how failure is recorded.
- **Simulated failure code is test infrastructure, not production logic**:
  - The `payload.contains("fail")` check exists only to deterministically test retries.
- **Atomic `UPDATE ... WHERE ...` with row-count checking is a race-safe pattern**:
  - `repository.claim(...) == 1` means this worker successfully owns the task.
- **Exponential backoff prevents retry storms**:
  - Doubling delays with a cap balances responsiveness and system stability.
- **Spring Data JPA concepts clarified**:
  - `@Param` binds method arguments to named query parameters safely.
  - `:id` and `:now` are placeholders, not string substitution.
  - `@Modifying` marks queries that change data and return affected row counts.
  - `Pageable` enforces bounded result sets and translates to SQL `LIMIT/OFFSET`.

### Decisions made
- Added a `DEAD` task status to represent terminal failure explicitly.
- Defined `FAILED` as retryable only when `nextRunAt` is present and in the future.
- Chose to increment `attemptCount` during task claiming (start of an attempt).
- Treated `nextRunAt == null` for `FAILED` as an inconsistent state, not normal behavior.
- Accepted that simulated failure logic is temporary and should be removed for production.
- Adopted a “verify-by-behavior” mindset instead of trusting code blindly.

### Problems / blockers
- Discovered multiple subtle logic issues in retry eligibility and attempt counting.
- Noticed mismatches between domain invariants and repository queries.
- Realized that some earlier code paths bypassed domain methods, causing state drift.
- Experienced loss of confidence due to uncovering mistakes, requiring recalibration of trust.

### Open questions / next steps
- Add automated tests to lock in retry invariants and prevent regressions.
- Consider adding assertions (e.g., `attemptCount > 0` on success).
- Decide whether to rename retry states to more human-friendly terms in the API.
- Continue to the next lesson with a focus on observability, testing, or scaling patterns.
