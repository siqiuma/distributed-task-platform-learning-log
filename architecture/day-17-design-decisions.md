## Worker Execution Model & Design Decisions
### Why a single worker thread per instance?

Each worker instance runs a single long-polling loop against SQS.  
This is intentional.

SQS long polling blocks efficiently until messages arrive, so additional
threads would provide no throughput benefit and would increase resource
usage and shutdown complexity.

Horizontal scaling is achieved by running multiple worker instances,
not by increasing threads within a single instance.
### Why long polling instead of frequent polling?

Workers use SQS long polling (`waitTimeSeconds=20`) to block until messages
arrive or a timeout expires.

This:
- reduces CPU usage
- lowers SQS API call cost
- minimizes delivery latency
- avoids busy-waiting loops

The worker remains idle when there is no work and wakes immediately
when tasks are available.
### Why the database is the source of truth (not SQS)

SQS provides delivery and wake-up semantics, but task state lives in the database.

Each task is claimed atomically via a conditional UPDATE
(`ENQUEUED → PROCESSING`).  
Only one worker can successfully claim a task, even with multiple workers
consuming the same queue.

This guarantees:
- at-least-once delivery
- no double execution
- safe retries
- correct recovery after crashes
