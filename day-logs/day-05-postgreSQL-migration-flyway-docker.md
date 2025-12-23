## Day 5 – Lesson 5: PostgreSQL Migration + Flyway + Docker (Why We Need Them)

### What I worked on
- Migrated the project from H2 (in-memory) to PostgreSQL for a more production-like setup.
- Added the PostgreSQL JDBC dependency to the project.
- Installed/used Docker to run a local PostgreSQL instance (so I don’t have to install Postgres directly on my machine).
- Created a dedicated config file `application-postgres.properties` and ran the app with the `postgres` profile.
- Introduced Flyway database migrations and created the first migration script `V1__init.sql` to create the `tasks` table.
- Tested end-to-end by creating tasks via API and verifying persisted rows from PostgreSQL using SQL queries.

### What I learned
- **H2 vs PostgreSQL**
  - H2 can auto-create tables based on JPA entities (convenient for learning), but it’s not a true production environment.
  - PostgreSQL behaves more like real production databases: schema management, types, locking behavior, and persistence across restarts.
- **Why we needed a SQL migration file after switching to Postgres**
  - With Postgres (and production-like settings), we want the schema to be explicitly managed and repeatable.
  - Relying on Hibernate auto-DDL in production is risky because it can cause unintended schema changes.
  - Flyway provides a controlled way to create/update schema reliably across environments.
- **What Flyway is doing**
  - Flyway runs versioned migration scripts (like `V1__init.sql`) to bring the database schema to the expected version.
  - It tracks what has already been applied using a metadata table (so it does NOT rerun old migrations every app start).
- **Why migrations have multiple versions**
  - In real systems, schema changes are incremental: add a column, create an index, backfill data, etc.
  - Each change becomes a new migration (V2, V3…) so environments can upgrade step-by-step consistently.
  - A single “big SQL file” becomes hard to maintain and unsafe when the schema evolves over time.
- **Production data safety**
  - We do NOT rerun `V1` every time; Flyway records applied migrations, so production data is preserved.
  - This avoids “schema resets” that would destroy existing data.
- **Where Postgres “lives”**
  - PostgreSQL is not in-memory by default; it stores data on disk.
  - When running in Docker, the DB is inside a container and can persist via Docker volumes (depending on config).
- **What Docker’s role is**
  - Docker is acting like a lightweight “packaged environment” so I can run Postgres locally without manual installation.
  - It helps reproduce the same DB setup reliably on any machine.
  - It’s OK not to use Docker, but then I must install and manage Postgres myself (versions, config, cleanup).
- **Work environment comparison (Oracle + Ansible Tower)**
  - My work setup sounds like “migration management” is done outside the app:
    - SQL is stored in a separate repository.
    - Deployments run schema SQL via Ansible Tower across environments.
  - That’s a valid alternative approach to Flyway:
    - Instead of app-managed migrations, schema changes are managed by an ops/database deployment pipeline.

### Decision(s) made
- Use PostgreSQL for a more realistic “production-like” development environment.
- Use Flyway migrations to manage schema changes safely and repeatably.
- Use Docker for local Postgres to reduce machine-specific setup issues and keep the environment reproducible.
- Prefer explicit schema migrations over Hibernate auto-DDL when simulating real-world backend development.

### Problems / blockers
- Encountered multiple environment/tooling issues during migration:
  - IDE warnings about SQL files (data source not configured) — resolved by configuring/ignoring IDE assistance.
  - Configuration warning for `spring.flyway.enabled` until Flyway starter dependency was correctly included.
  - Missing `tasks` table due to schema validation before migration was applied.
  - Missing `psql` client on the machine (`psql: command not found`) until installing client tools or using alternatives.

### Open questions / next steps
- Confirm Docker volume configuration to ensure Postgres data persists across container restarts (to better simulate production).
- Add future migrations (V2, V3…) for schema evolution (indexes, new columns like `attemptCount`, etc.).
- Consider adding a short `docs/db.md` explaining:
  - how to start Postgres
  - how migrations work
  - how to verify data quickly
- Continue Lesson 5 by adding indexing and performance-related improvements (e.g., index on `status, created_at` for worker queries).
