### Day 10 Lesson 9 – Repository-level tests (JPA + Postgres)

### What I worked on
- Created `TaskRepositoryTest.java` under `src/test/java/com/siqiu/distributedtaskplatform/task`
- Wrote repository-level tests against Postgres (Testcontainers) to verify DB behavior matches domain expectations
- Verified the `claim(...)` update query returns correct row counts (1 for first claim, 0 for second claim)
- Verified `findTop5Eligible(...)` filtering logic:
  - Includes `FAILED` tasks when `nextRunAt <= now`
  - Excludes `FAILED` tasks when `nextRunAt > now`
  - Excludes `DEAD` tasks always
- Used `PageRequest.of(0, 5)` to match repository signature with `Pageable` and control result size deterministically
- Used `saveAndFlush(...)` to ensure test data is written before queries run

### What I learned
- `@Modifying` update queries must run inside a transaction in tests, otherwise JPA throws “Executing an update/delete query” errors
- A successful claim pattern relies on row-count semantics:
  - `updatedRows == 1` means “I claimed it”
  - `updatedRows == 0` means “someone else already claimed it”
- `FAILED` retry eligibility is not just status-based; it depends on time (`nextRunAt`) relative to `now`
- Repository tests are different from domain tests: they validate real query correctness and database semantics (ordering, filters, pagination)

### Problems / blockers
- Got `InvalidDataAccessApiUsageException: Executing an update/delete query` because the update query was not executed within a transactional test context
- Initially saw the task status still `PENDING` after `claim(...)` because the test was asserting the old entity instance instead of reloading/observing DB state
- `findTop5Eligible(...)` test initially returned an empty list because `nextRunAt` was not due yet (debugged by printing `nextRunAt` and comparing to `now`)

### Decision(s) made
- Use Postgres (Testcontainers) for repository tests so we can rely on Postgres behavior (partial indexes, timestamp types, query semantics)
- Use `saveAndFlush` + fresh reads to avoid false assumptions from persistence context caching
- Keep repository tests focused on query correctness and row-count semantics, not worker behavior

### Open questions / next steps
- Move to: worker behavior tests (ensure tasks aren’t processed twice, retries only occur when due, and `DEAD` tasks stop permanently)
