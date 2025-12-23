### 1. Why I introduced `TaskWorkerTx` (transaction helper)

I introduced a separate `TaskWorkerTx` component to clearly define **transaction boundaries** and avoid Spring’s self-invocation limitation with `@Transactional`. In Spring, transactional behavior is applied via proxies, which means calling a `@Transactional` method from within the same class may bypass the proxy and silently disable the transaction.

By moving all database state transitions (claiming a task, marking it succeeded/failed, reading payload) into a dedicated helper bean, each operation is guaranteed to execute inside a properly scoped transaction. This separation makes the code easier to reason about and ensures that each database interaction is atomic, short-lived, and correctly managed by Spring.

Conceptually:
- `TaskWorker` is responsible for orchestration and scheduling.
- `TaskWorkerTx` is responsible for **atomic database changes**.

This pattern avoids accidental long-running transactions and makes concurrency behavior explicit and predictable.

#### Spring has a very important rule:
❌ @Transactional does not work reliably when you call a method from the same class.
This is called the self-invocation problem.

#### Example of what does not work as expected:
```java
@Component
class TaskWorker {

    @Transactional
    public void claim() { ... }

    public void run() {
        claim(); // ❌ transaction may NOT be applied
    }
}
```
Spring applies @Transactional using proxies.
Calling a method inside the same class bypasses the proxy.

#### Why TaskWorkerTx fixes this

By moving transactional methods into another bean:
```java
@Component
class TaskWorker {
    private final TaskWorkerTx tx;

    void run() {
        tx.claim(id); // ✅ goes through Spring proxy
    }
}
```
### 2. What “perform processing outside any transaction” means

I learned that long-running work (such as sleeping, computation, or calling external services) should **never be performed inside a database transaction**. Holding a transaction open during processing causes database locks to be held unnecessarily, leading to lock contention, timeouts (especially in H2), and poor scalability.

The correct pattern is:
1. **Claim the task** in a short transaction (e.g., `PENDING → PROCESSING`)
2. **Perform processing outside any transaction** (no database locks held)
3. **Persist the result** in another short transaction (e.g., `PROCESSING → SUCCEEDED/FAILED`)

In practice, this means:
- Transactions are kept as short as possible
- The database is only locked for milliseconds, not seconds
- Concurrent API requests (like cancel) can proceed safely
- The system behaves predictably under concurrency

This separation of concerns is critical for building production-safe background workers.
What happens if you process inside a transaction
##### Bad pattern:
```java
@Transactional
public void process() {
    claimTask();

    Thread.sleep(3000); // ❌ holding DB locks

    markSucceeded();
}
```
While sleeping:
- DB row/table may be locked
- cancel requests block
- other workers block
- H2/Postgres may time out
##### Correct pattern
```java
// 1) short transaction
tx.claim(id);

// 2) NO TRANSACTION
Thread.sleep(3000); // processing

// 3) short transaction
tx.markSucceeded(id);
```
