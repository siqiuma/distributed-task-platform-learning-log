## 🗺 System Diagram — Distributed Task Platform

### Phase 1: Core API (Current)

```mermaid
flowchart LR
    Client[Client / curl / UI]
    API[Task API Service<br/>Spring Boot 4]
    DB[(Task Store<br/>H2 now / Postgres later)]

    Client -->|HTTP REST| API
    API -->|JPA| DB
