# Transaction Propagation and Isolation

> **Source:** Lecture 34 (`Notes_Of_Aditya/Lecture 34/code/transactionDemo`)
> **Prerequisite:** [42_Transactions and ACID](42_Transactions%20and%20ACID.md)

## Why this topic matters
Senior-level Spring interviews go beyond "use `@Transactional`": REQUIRED vs REQUIRES_NEW, `UnexpectedRollbackException`, dirty/non-repeatable/phantom reads, and choosing an isolation level.

---

## 1. `@Transactional` attributes and defaults
```java
@Transactional(propagation = Propagation.REQUIRED,      // default
               isolation   = Isolation.DEFAULT,          // default → DB's level
               timeout     = 5,                          // seconds (default: infrastructure default)
               readOnly    = false,                      // default
               rollbackFor = IOException.class)          // default: RuntimeException + Error only
```
All of it is applied through the **proxy**, so calls must go through the bean (no self-invocation).

## 2. Propagation: what to do if a transaction already exists?
| Propagation | Transaction exists | No transaction |
|---|---|---|
| **REQUIRED** (default) | Join | Create |
| **REQUIRES_NEW** | **Suspend** it, create a new independent one | Create |
| SUPPORTS | Join | Run without |
| MANDATORY | Join | **Throw** |
| NOT_SUPPORTED | Suspend, run without | Run without |
| NEVER | **Throw** | Run without |
| NESTED | **Savepoint** inside it | Create (like REQUIRED) |
"Suspend" ≠ commit. The outer transaction just pauses and resumes afterwards.

### REQUIRED: one shared physical transaction
```java
@Transactional
public void checkout() {
    orderService.createOrder();         // REQUIRED → joins
    inventoryService.reserveStock();    // joins
    paymentService.createPaymentRecord();
}
```
- They share one connection, one persistence context, and **one fate**: an outer failure after inner success rolls **everything** back.
- An inner method's own `isolation`/`timeout` is **ignored**; the outer transaction's settings win (Spring only validates this if `validateExistingTransaction=true`).

**Lecture code:**
```java
@Transactional
public void placeOrder(Order order) {          // OrderService: starts Tx A
    orderRepository.save(order);
    paymentAuditService.audit(order);          // REQUIRED → joins Tx A
}

@Transactional(propagation = Propagation.REQUIRED, isolation = Isolation.REPEATABLE_READ)
public void audit(Order order) {               // ⚠️ REPEATABLE_READ is IGNORED: it joins Tx A
    paymentAuditRepository.save(new PaymentAudit(order.getAmount(), order.getId(), true));
}
```
If anything fails after the audit, **the audit row is rolled back too**. Change `audit` to `REQUIRES_NEW` to keep the audit record (see below).

### ⚠️ `UnexpectedRollbackException`
```java
@Transactional
public void placeOrder() {
    orderRepository.save(new Order());
    try { inventoryService.reserveStock(); }          // REQUIRED, throws RuntimeException
    catch (InsufficientStockException e) { log.warn("failed"); }
}   // tries to commit → Tx is ROLLBACK-ONLY → rollback + UnexpectedRollbackException
```
The inner interceptor marked the **shared** transaction rollback-only. Catching the Java exception doesn't undo that mark.
Fixes: let it propagate; use `REQUIRES_NEW`/`NESTED` for the inner call; or make the inner method not transactional (or use `noRollbackFor`) if failure is acceptable.

### REQUIRES_NEW: independent physical transaction
```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void recordAttempt() { auditRepository.save(new AuditRecord()); }
```
```
placeOrder()  Tx A ──► suspend A ──► Tx B (own connection + persistence context) commit/rollback ──► resume A
```
| Scenario | Result |
|---|---|
| Inner commits, outer throws | **Audit kept**, order rolled back |
| Inner throws and outer catches it | Inner rolled back, **outer can still commit** (it isn't marked rollback-only) |
Use for audit or error logs that must survive the main failure.
Costs: a **second DB connection** while the first is held (pool exhaustion or deadlock risk under load). Tx B **can't see Tx A's uncommitted rows**. And the inner call **must go through another bean's proxy** (self-calls don't start a new transaction).

### MANDATORY / NESTED / others
- **MANDATORY**: "only callable inside a caller's transaction" (e.g. `reserveStock` as a building block).
- **NESTED**: one physical transaction with a **savepoint**. Inner failure → `ROLLBACK TO SAVEPOINT`, outer continues; outer rollback undoes everything. Needs JDBC savepoint support. It works with `DataSourceTransactionManager`. With `JpaTransactionManager`, `nestedTransactionAllowed` defaults to `false`, and even when enabled, the savepoint covers only the JDBC connection, **not** the EntityManager's cached entities.
- **SUPPORTS**: read helpers that work either way. **NOT_SUPPORTED/NEVER**: must run outside a transaction.

## 3. Isolation: what can concurrent transactions see?
### Anomalies
| Anomaly | What happens |
|---|---|
| **Dirty read** | T2 reads T1's **uncommitted** change; T1 rolls back, so T2 used data that never existed |
| **Non-repeatable read** | T1 reads a row twice and gets **different committed values** (T2 updated it in between) |
| **Phantom read** | T1 re-runs `WHERE salary > 100000` and sees **new rows** T2 inserted |
| (Lost update) | Two read-modify-write cycles overwrite each other |

### Levels
| Isolation | Dirty | Non-repeatable | Phantom |
|---|---|---|---|
| READ_UNCOMMITTED | possible | possible | possible |
| **READ_COMMITTED** (PostgreSQL, Oracle, SQL Server default) | ✅ prevented | possible | possible |
| **REPEATABLE_READ** (**MySQL InnoDB default**) | ✅ | ✅ | allowed by the standard (InnoDB's MVCC + gap locks prevent most) |
| SERIALIZABLE | ✅ | ✅ | ✅ (result equals some serial order) |
`Isolation.DEFAULT` = use the database's own default. It isn't a new level.
Implementation varies: locks (row/range/gap), **MVCC** snapshots, predicate locks, abort-and-retry. Stronger isolation means more blocking, deadlocks, and retries.

**Choose the weakest level that keeps your business rule correct.** Often **optimistic locking** (`@Version`) or **pessimistic locking** (`@Lock(LockModeType.PESSIMISTIC_WRITE)` → `SELECT … FOR UPDATE`) solves the specific race (e.g. double withdrawal) better than raising isolation globally.

## 4. `readOnly` and `timeout`
- `readOnly = true`: a hint. Hibernate skips dirty checking and flush (less memory/CPU), and some drivers/DBs route to replicas or optimize. It doesn't strictly forbid writes.
- `timeout`: max seconds for the transaction. Also consider query, lock, and pool timeouts.

---

## Interview Q&A
**Q1. Default propagation and isolation?**
REQUIRED; DEFAULT (the database's level: MySQL = REPEATABLE READ, PostgreSQL = READ COMMITTED).

**Q2. REQUIRED vs REQUIRES_NEW?**
REQUIRED joins the caller's transaction (shared fate). REQUIRES_NEW suspends it and runs an independent transaction that commits or rolls back on its own.

**Q3. What is `UnexpectedRollbackException`?**
An inner REQUIRED method threw, marking the shared transaction rollback-only. The outer method caught the exception and tried to commit, so Spring rolls back and throws this.

**Q4. When would you use REQUIRES_NEW?**
Audit/failure logs, sequence or ID allocation, notifications that must persist even if the main transaction fails.

**Q5. REQUIRES_NEW vs NESTED?**
REQUIRES_NEW uses a separate physical transaction and connection. NESTED uses a savepoint in the same transaction (an outer rollback undoes it too).

**Q6. Explain dirty, non-repeatable, and phantom reads.**
Reading uncommitted data / the same row changing between reads / the set of matching rows changing between reads.

**Q7. Inner method declares SERIALIZABLE but joins an outer READ_COMMITTED transaction. Which applies?**
The outer one: joined transactions keep their original characteristics.

**Q8. How do you prevent two concurrent withdrawals from overdrawing an account?**
Pessimistic lock (`SELECT … FOR UPDATE`), optimistic `@Version` with retry, an atomic conditional UPDATE (`… WHERE balance >= :amt`), or SERIALIZABLE with retry.

## Quick revision
- REQUIRED = join or create (shared fate); REQUIRES_NEW = suspend + independent; MANDATORY = must exist; NESTED = savepoint.
- Caught inner exception + REQUIRED = `UnexpectedRollbackException`.
- Joined transactions ignore the inner isolation/timeout (lecture's REPEATABLE_READ audit is ignored).
- RU < RC < RR < SERIALIZABLE; MySQL defaults to RR, PostgreSQL to RC.
- Prefer targeted locking (`@Version`, `FOR UPDATE`) over blanket SERIALIZABLE.
