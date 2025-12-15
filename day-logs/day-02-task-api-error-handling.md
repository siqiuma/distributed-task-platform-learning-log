## Day 2 – Lesson 2: Task API Completion & Robust Error Handling

### What I worked on
- Continued building the core Task API vertical slice
- Implemented proper error handling for missing resources (404 Not Found)
- Added domain-specific exception `TaskNotFoundException`
- Introduced a global exception handler using `@RestControllerAdvice`
- Implemented task cancellation via `PUT /tasks/{id}/cancel`
- Enforced valid task state transitions inside the `Task` entity
- Introduced `InvalidTaskStateException` for illegal state changes
- Mapped invalid state transitions to HTTP `409 Conflict`
- Verified behavior using `curl` with headers (`-i`) to confirm status codes
- Validated that repeated cancel requests are idempotent and return correct responses

### What I learned
- REST APIs should communicate failures using meaningful HTTP status codes, not generic 500 errors
- `404 Not Found` represents a missing resource, not a server failure
- `409 Conflict` is the correct status for valid requests that conflict with current resource state
- Domain rules (such as allowed state transitions) belong inside the entity, not the controller
- `@ExceptionHandler` + `@ResponseStatus` translate Java exceptions into correct HTTP semantics
- `curl` does not show HTTP status codes unless headers are explicitly requested

### Decision(s) made
- Use custom domain exceptions instead of generic exceptions
- Centralize error-to-HTTP mapping using a global exception handler
- Use `PUT` for canceling tasks to reflect idempotent state updates
- Keep entity state changes explicit and protected via domain methods (no generic setters)

### Problems / blockers
- Initially thought HTTP 409 was not being returned because `curl` only showed the response body
- Resolved by using `curl -i` to inspect response headers and confirm status code

### Open questions / next steps
- Add handling for other invalid transitions (e.g., cancel after processing)
- Consider returning structured error objects instead of simple message maps
- Begin Lesson 3: task processing simulation (worker / scheduler)
