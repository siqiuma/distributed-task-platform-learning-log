## Day 9 – Lesson 8 Testing Foundation with Testcontainers + MockMvc (Domain Level Tests)

### What I worked on
- Switched project baseline to **Spring Boot 3.1.5** (stable release) while keeping the same architecture and functionality.
- Set up a **test profile** so running `./mvnw test` uses an isolated environment.
- Added **Testcontainers-based PostgreSQL** support for integration tests (real DB, ephemeral container).
- Created/updated the following test and configuration files:
  - `TestcontainersConfig` (Testcontainers + datasource wiring for tests)
  - `application-test.properties` (test profile configuration)
  - `PostgresContainerSmokeTest` (sanity check that Postgres container + Flyway + JPA boot correctly)
  - `TaskDomainTest` (fast unit tests for domain invariants/state transitions)
  - `TaskApiIntegrationTest` (API-level integration tests using MockMvc)
- Fixed Maven/Testcontainers dependency issues until `./mvnw test` ran cleanly.
- Ran and verified successful test execution with logs showing:
  - Postgres container startup
  - Flyway migrations applying (V1/V2/V3)
  - Spring context boot + repository wiring
  - API tests passing end-to-end

### What I learned
- **Why MockMvc is useful even when I “don’t remember using Spring MVC”:**
  - My controllers are part of the **Spring MVC web layer** because I use annotations like `@RestController`, `@PostMapping`, `@GetMapping`, etc.
  - **MockMvc** simulates real HTTP requests to my controller without starting a real server (no port, no Tomcat networking), which makes API tests fast and reliable.
- **Why Testcontainers is valuable:**
  - H2 can behave differently from PostgreSQL (SQL features, indexes, data types, locking), so a real Postgres test DB catches issues earlier.
  - Testcontainers gives a clean database per test run, which keeps tests repeatable and avoids “works on my machine” drift.
- **How the test layers map to confidence:**
  - Domain tests lock down business rules (fast feedback).
  - Repository/DB integration tests validate query semantics and migrations against real Postgres.
  - API integration tests validate the “contract” (status codes + JSON response shape) through the controller.

### Decision(s) made
- Use **Spring Boot 3.1.5** as the stable baseline version for this project going forward.
- Keep **MockMvc** as the primary tool for API integration tests (fast, no real server).
- Use **Postgres via Testcontainers** for integration tests where database behavior matters, instead of relying on H2.

### Problems / blockers
- Testcontainers dependencies initially failed to resolve (missing versions / incorrect artifact IDs).
- Tests sometimes ran against H2 by default until the **test profile** and container configuration were correctly applied.
- Flyway migration compatibility problems surfaced when running migrations on different DB engines (H2 vs Postgres), reinforcing the need to test with Postgres.

### Open questions / next steps
- **Repository-level tests** (prove `claim(...)` row-count semantics and `findTop5Eligible(...)` eligibility rules)
- **Worker behavior tests** (verify retries and DEAD behavior without flaky sleeps)
- **Observability improvements** (structured decision logs for claim/retry/DEAD transitions)
- **Interview storytelling** (turn this work into resume bullets + system-design narrative)
