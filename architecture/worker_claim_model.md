```mermaid
flowchart LR
    Client[Client]
    API[Task API]
    DB[(Database)]
    Worker[Task Worker]
    Work[Task Processing]

    Client --> API
    API --> DB

    Worker --> DB
    Worker -->|claim task| DB
    Worker --> Work
    Worker -->|update result| DB
```
**Worker execution model**

1. Worker polls database for PENDING tasks
2. Worker atomically claims a task (PENDING → PROCESSING)
3. Task processing runs outside any transaction
4. Worker updates final state (PROCESSING → SUCCEEDED / FAILED)

Optimistic locking (`@Version`) is used as a safety net to detect concurrent updates.
