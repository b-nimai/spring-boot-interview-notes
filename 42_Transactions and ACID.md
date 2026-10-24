# Transactions and ACID

> **Source:** Lecture 33 (`Notes_Of_Aditya/Lecture 33/code/transactionDemo`)
> **Next:** [43_Transaction Propagation and Isolation](43_Transaction%20Propagation%20and%20Isolation.md) · **Related:** [38_Hibernate Internals](38_Hibernate%20Internals%20-%20Persistence%20Context%2C%20Dirty%20Checking%20and%20Flush.md)

## Why this topic matters
"Explain ACID" and "How does `@Transactional` work?" are near-certain interview questions. A money transfer is the classic example.

---

## 1. What a transaction is
A **unit of work** that must be applied **completely or not at all**. It's defined by the business operation, not by the number of SQL statements.
```
Transfer ₹1,000 A → B
UPDATE account SET balance = 9000 WHERE id = 1;   -- debit
UPDATE account SET balance = 6000 WHERE id = 2;   -- credit
Total before = total after = ₹15,000
```

## 2. ACID
| Property | Guarantee | Transfer example |
|---|---|---|
| **Atomicity** | All or nothing | Debit without credit is impossible; a failure rolls back the debit |
| **Consistency** | Valid state → valid state (constraints and rules hold) | No negative balance; money isn't created or destroyed |
| **Isolation** | Concurrent transactions don't corrupt each other | Two withdrawals don't overwrite each other ([43](43_Transaction%20Propagation%20and%20Isolation.md)) |
| **Durability** | Committed = survives crashes/restarts | After "success", a power cut doesn't lose it (redo log/WAL) |
Consistency depends on rules you define (constraints, app checks); the transaction guarantees they aren't left half-applied.

## 3. Lifecycle: begin, commit, rollback
- **Begin**: start the unit (JDBC: `setAutoCommit(false)`).
- **Commit**: make the changes permanent and visible.
- **Rollback**: discard uncommitted changes (the DB uses its undo log; you don't write reverse UPDATEs).
JDBC default is **auto-commit = true**: every statement is its own transaction.

## 4. Same transaction, four APIs
```
@Transactional service  →  Spring PlatformTransactionManager (JpaTransactionManager)
                        →  JPA EntityTransaction / Hibernate Session.Transaction
                        →  JDBC Connection (setAutoCommit / commit / rollback)
                        →  Database transaction
```
```java
// Raw JDBC
Connection c = dataSource.getConnection();
try { c.setAutoCommit(false); /* SQL */ c.commit(); }
catch (Exception e) { c.rollback(); throw e; }
finally { c.close(); }

// JPA (resource-local)
EntityTransaction tx = em.getTransaction();
try { tx.begin(); /* persist… */ tx.commit(); } catch (Exception e) { tx.rollback(); throw e; }

// Spring: declarative
@Transactional
public void transfer(...) { /* business logic only */ }
```
Spring doesn't replace the DB transaction. It **drives** it for you through an AOP proxy.

## 5. How `@Transactional` works
```
Caller → [TransactionInterceptor proxy] → begin (or join) → method → commit | rollback → release
```
Use `org.springframework.transaction.annotation.Transactional` (supports `readOnly`, `propagation`, `isolation`, `timeout`, `rollbackFor`).
- **Method-level** or **class-level** (a default for public methods; method annotations override it).
- Put it on the **service layer**: one business use case = one transaction spanning several repositories.

### Rollback rules
| Exception escaping the method | Default |
|---|---|
| `RuntimeException` (unchecked) / `Error` | **Rollback** |
| Checked `Exception` | **Commit** (unless `rollbackFor = X.class`) |
| Caught inside and swallowed | **Commit** (Spring never saw it) |
| Return values (`false`, `null`, `badRequest()`) | Irrelevant: commit |

### Proxy pitfalls (why `@Transactional` "doesn't work")
- **Self-invocation** (`this.transfer()`) bypasses the proxy, so there's no transaction.
- `private` (and with CGLIB, `final`) methods aren't intercepted.
- The object isn't a Spring bean (`new Service()`).
- A checked exception was thrown (it commits), or the exception was caught.

## 6. Lecture code: transfer with forced failure
```java
@Entity
public class Account {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    private String name;
    private BigDecimal balance;                         // BigDecimal for money, never double

    public void debitAccount(BigDecimal amount) {       // business rules live in the entity
        if (amount == null || amount.signum() <= 0) throw new IllegalArgumentException("Amount must be positive");
        if (balance.compareTo(amount) < 0) throw new RuntimeException("Insufficiemt Balance");
        balance = balance.subtract(amount);
    }
    public void creditAccount(BigDecimal amount) { /* validate */ balance = balance.add(amount); }
}

@Service
public class TransferService {
    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) throws Throwable {
        Account from = accountRepository.findById(fromId).orElseThrow(() -> new RuntimeException("User not found"));
        Account to   = accountRepository.findById(toId).orElseThrow(() -> new RuntimeException("User not found"));

        from.debitAccount(amount);
        accountRepository.saveAndFlush(from);          // UPDATE sent to DB now (flush ≠ commit)
        to.creditAccount(amount);
        accountRepository.saveAndFlush(to);            // UPDATE sent
        transferRepository.save(new TransferRecord(fromId, toId, amount, LocalDate.now()));
        transferRepository.flush();                    // INSERT sent

        throw new RuntimeException("Some Error Occured"); // demo: simulate a failure after everything
    }
}
```
**Expected result** (this project needs MySQL, so it wasn't run here): `show-sql` prints two UPDATEs and one INSERT, yet after the 500 error the **balances are unchanged and no transfer record exists**. The `RuntimeException` triggered a rollback of the whole transaction. That's atomicity in action, and it proves **flush ≠ commit**.
Remove the `@Transactional` (or throw a checked exception) and the debit stays committed: money disappears.

> ⚠️ Lecture notes: `throws Throwable` in the controller and service is unnecessary. The demo always throws (by design). Insufficient balance should be a custom `InsufficientBalanceException extends RuntimeException` → 409/422 via `@RestControllerAdvice`. Concurrent transfers on the same account still need **locking** (`@Lock(PESSIMISTIC_WRITE)` or `@Version` optimistic locking). Isolation alone doesn't stop lost updates in every DB.

---

## Interview Q&A
**Q1. Explain ACID with an example.**
Bank transfer. Atomicity: debit and credit together. Consistency: totals and constraints stay valid. Isolation: concurrent transfers don't interfere. Durability: committed changes survive crashes.

**Q2. How does `@Transactional` work internally?**
A Spring AOP proxy with `TransactionInterceptor` asks the `PlatformTransactionManager` to begin or join, invokes the method, then commits or rolls back based on the outcome and rollback rules.

**Q3. Which exceptions cause rollback by default?**
Unchecked (`RuntimeException`) and `Error`. Checked exceptions commit unless configured with `rollbackFor`.

**Q4. Why put `@Transactional` on the service layer?**
The service method is the business unit of work, spanning multiple repository calls that must succeed or fail together.

**Q5. Flushed but then an exception: is the data saved?**
No. Flush sends SQL within the open transaction; rollback discards it.

**Q6. Why might `@Transactional` be ignored?**
Self-invocation, a private/final method, a non-bean object, a checked or caught exception, or the wrong annotation/transaction manager.

**Q7. What is auto-commit?**
The JDBC default: each statement commits immediately. Transactions disable it until commit or rollback.

**Q8. Why `BigDecimal` for money?**
`double` can't represent decimal fractions exactly (0.1 + 0.2 ≠ 0.3).

## Quick revision
- Transaction = all-or-nothing business unit; ACID guarantees.
- begin → commit | rollback; JDBC auto-commit is on by default.
- `@Transactional` (Spring) on services; proxy-based (self-invocation pitfall).
- Rollback on RuntimeException/Error; checked exceptions commit unless `rollbackFor`.
- Flush ≠ commit (lecture demo proves it).
