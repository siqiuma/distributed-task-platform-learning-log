## Day 5 – Lesson 3: Background Task Processing & Worker Model

### What I worked on
- Implemented a background worker using Spring’s `@Scheduled` mechanism
- Extended the task lifecycle to include `PROCESSING`, `SUCCEEDED`, and `FAILED`
- Added explicit state transition methods to the `Task` entity to enforce valid workflows
- Implemented polling-based task processing that runs independently of HTTP requests
- Added structured logging to observe asynchronous task execution
- Simulated task execution and intentional failures to validate behavior
- Verified task state changes and timestamps using API calls and logs

### What I learned
- Backend systems often continue work after the HTTP request has completed
- `@Scheduled` enables time-based background execution without user interaction
- Polling workers pick up tasks based on scheduler ticks, not per-task timers
- Asynchronous systems are understood primarily through logs, not return values
- Entity-level state transition methods help protect domain invariants
- `@Transactional` ensures consistent updates during background processing
- Task timestamps reflect when persistence occurs, not when scheduling intent begins

### Decision(s) made
- Use a polling worker model for simplicity and clarity before introducing queues
- Keep task processing logic outside controllers to separate HTTP and background concerns
- Use logging as the primary tool for understanding and debugging async behavior
- Accept non-deterministic timing (e.g., ~3s vs ~5s) as expected polling behavior

### Problems / blockers
- Initially expected task completion timing to be exactly aligned with scheduler delay
- Resolved by understanding scheduler tick alignment and JPA flush timing

### Open questions / next steps
- Explore retry logic for failed tasks
- Add concurrency safeguards and locking strategies
- Transition from polling to queue-based processing in a future lesson
- Begin Lesson 4: robustness, concurrency, and persistence considerations

***Domain invariants** are the rules that must always be true for your core business objects, no matter where the change comes from (REST API, worker, scheduler, tests, future Kafka consumer, etc.).*
- Data invariants (Task record must be valid)
- Lifecycle invariants (Task status rules)
- Time invariants (timestamps must make sense)
- API/behavior invariants (how your API should behave)
