## Day 5 – Lesson 5: Indexing and Performance Awareness

### What I worked on
- Reviewed the worker’s hot-path query (`findTop5ByStatusOrderByCreatedAt`) to identify performance-critical access patterns.
- Confirmed that the initial composite index `(status, created_at)` already existed in the V1 schema migration.
- Added a refined **partial index** via Flyway (V2 migration) to optimize the most common worker case (`status = 'PENDING'`).
- Ran the application using the Postgres profile to ensure Flyway migrations executed against Postgres instead of H2.
- Connected to Postgres using `psql` to inspect tables, indexes, and migration history.
- Verified the existence and definition of the new partial index using Postgres system catalogs.
- Used `EXPLAIN` to confirm that Postgres can use the partial index for the worker polling query.

### What I learned
- **Indexing should be driven by query patterns**, not added blindly.
  - The worker repeatedly queries `WHERE status = 'PENDING' ORDER BY created_at LIMIT N`, making it a clear hot path.
- **Composite vs partial indexes**
  - A composite index `(status, created_at)` is general-purpose and supports multiple statuses.
  - A partial index (`WHERE status = 'PENDING'`) is smaller and faster for the worker because it only indexes relevant rows.
- **Why partial indexes are powerful in Postgres**
  - They reduce index size and IO by excluding irrelevant rows.
  - They are especially effective when most rows are not in the queried state.
- **How to verify indexes correctly**
  - Structural verification using `pg_indexes` confirms that the index exists and is defined as expected.
  - Behavioral verification using `EXPLAIN` confirms that the query planner can choose the index.
- **Why this matters for background workers**
  - Polling queries run frequently and scale poorly without proper indexing.
  - Index design directly affects how well workers scale as data volume grows.
- **Flyway and database specificity**
  - Advanced SQL features (like partial indexes) are database-specific and must be run against the correct DB.
  - Accidentally running Postgres-specific migrations against H2 results in syntax errors, reinforcing the need for correct profiles.

### Decision(s) made
- Keep the original composite index from V1 for general querying.
- Add a Postgres-specific partial index in V2 to optimize the worker’s hot path.
- Always verify schema changes directly in the database rather than relying only on application behavior.

### Problems / blockers
- Initially encountered SQL syntax errors when the partial index migration was executed against H2 instead of Postgres.
- Resolved by ensuring the application and Flyway ran with the Postgres profile enabled.

### Open questions / next steps
- Decide whether additional partial or covering indexes are needed as new access patterns emerge.
- Document the reasoning behind existing indexes in the project README.
- Move on to Lesson 6, focusing on retries, failure handling, and idempotency.
