```mermaid
flowchart LR
    Client[Client / Producer]
    API[Task API Service<br/>Spring Boot 4]
    DB[(Task Store<br/>Postgres)]
    Q[(Task Queue<br/>Kafka / SQS / PubSub)]

    Client -->|HTTP REST| API
    API -->|persist task| DB
    API -->|enqueue task| Q

    subgraph Workers["Worker Fleet"]
        W1[Worker A]
        W2[Worker B]
        W3[Worker C]
    end

    Q -->|consume| W1
    Q -->|consume| W2
    Q -->|consume| W3

    W1 -->|update status| DB
    W2 -->|update status| DB
    W3 -->|update status| DB

    Scheduler[Scheduler Service<br/>Spring Boot 4]
    Scheduler -->|scan scheduled tasks| DB
    Scheduler -->|enqueue due tasks| Q
