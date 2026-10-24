# Optimistic and Pessimistic Locking and Idempotency

> **Part of:** Spring gap topics (63–71) · **Related:** [43_Isolation](43_Transaction%20Propagation%20and%20Isolation.md), [60_Concurrency](60_Concurrency%20-%20Synchronization%2C%20Locks%2C%20volatile%20and%20Atomics.md)

## Why this topic matters
"Two users book the last seat at the same time", "a payment request is retried", "double-click on Pay" are the most common **product-company scenario questions**, and JVM `synchronized` doesn't help once you run more than one instance.

---

## 1. The problem: lost update
```
T1: read stock = 1          T2: read stock = 1
T1: stock - 1 → write 0     T2: stock - 1 → write 0     → 2 items sold, stock says 0 (oversold)
```
With multiple app instances, **in-JVM locks are useless**. The coordination must happen in the **database** (or a distributed lock).

## 2. Optimistic locking (`@Version`)
Assume conflicts are rare. Don't lock; **detect** conflicts at write time.
```java
@Entity
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    private int stock;
    @Version private Long version;          // managed by Hibernate
}
```
Hibernate's UPDATE becomes:
```sql
UPDATE product SET stock = ?, version = 6 WHERE id = ? AND version = 5;
-- 0 rows updated → someone else changed it → OptimisticLockException
--                  (Spring: ObjectOptimisticLockingFailureException)
```
```java
@Retryable(retryFor = ObjectOptimisticLockingFailureException.class, maxAttempts = 3)   // Spring Retry
@Transactional
public void purchase(Long productId, int qty) {
    Product p = repo.findById(productId).orElseThrow();
    if (p.getStock() < qty) throw new OutOfStockException();
    p.setStock(p.getStock() - qty);          // version checked at flush/commit
}
```
- Fits **low-contention** data (profile edits, documents, admin updates).
- For REST, return the version as an **ETag** and require `If-Match`. Mismatch → **409 Conflict / 412 Precondition Failed**, so a user doesn't overwrite someone else's edit (the lost-update problem in UIs).
- On conflict: retry (re-read and re-apply) or report a 409 to the user.

## 3. Pessimistic locking (`SELECT … FOR UPDATE`)
Assume conflicts are likely. **Lock the row** while working.
```java
public interface AccountRepository extends JpaRepository<Account, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)                          // SELECT ... FOR UPDATE
    @QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findForUpdate(@Param("id") Long id);
}

@Transactional
public void transfer(Long from, Long to, BigDecimal amt) {
    Long first = Math.min(from, to), second = Math.max(from, to);   // consistent lock order → no deadlock
    Account a = repo.findForUpdate(first).orElseThrow();
    Account b = repo.findForUpdate(second).orElseThrow();
    ...                                                                // other transactions wait on these rows
}
```
| Mode | SQL | Meaning |
|---|---|---|
| `PESSIMISTIC_READ` | `FOR SHARE` | Others can read-lock, not write |
| `PESSIMISTIC_WRITE` | `FOR UPDATE` | Exclusive |
| `OPTIMISTIC_FORCE_INCREMENT` | — | Bump the version even without changes |
- Must run **inside a transaction**; the lock is held until commit/rollback, so keep transactions short.
- Risks: blocking, deadlocks (lock in a consistent order), lock timeouts.
- `SKIP LOCKED` (`FOR UPDATE SKIP LOCKED`) suits job queues where each worker grabs different rows.

## 4. Atomic conditional update (often the best)
```java
@Modifying
@Query("UPDATE Product p SET p.stock = p.stock - :qty WHERE p.id = :id AND p.stock >= :qty")
int decrementStock(@Param("id") Long id, @Param("qty") int qty);   // returns rows affected

if (repo.decrementStock(id, qty) == 0) throw new OutOfStockException();   // single atomic statement
```
No read-modify-write race, no explicit lock, very fast. Great for counters, stock, and seats.

## 5. Choosing
| Situation | Choose |
|---|---|
| Rare conflicts, user edits | **Optimistic** (`@Version`, ETag/If-Match) |
| Hot rows, high contention, must not fail (money moves) | **Pessimistic** (`FOR UPDATE`, ordered) or an atomic UPDATE |
| Simple counters/stock | **Atomic conditional UPDATE** |
| Uniqueness (one booking per seat) | **DB unique constraint** (the ultimate guard) |
| Cross-service / non-DB resource | Distributed lock (Redis `SET NX PX`, Redisson) **with fencing tokens**, or a design that avoids it |
**Unique constraints are your friend**: `UNIQUE(show_id, seat_no)` guarantees no double booking even if application checks race. Catch `DataIntegrityViolationException` and return 409.

## 6. Idempotency
An operation is **idempotent** if doing it multiple times has the **same effect** as doing it once.
| HTTP method | Idempotent? |
|---|---|
| GET, HEAD, PUT, DELETE | ✅ (by spec) |
| POST | ❌ (creates a new thing each time) |
| PATCH | Not guaranteed |
**Why it matters:** networks fail *after* the server processed the request. The client times out and retries, causing a double charge or duplicate order. Message queues also deliver **at-least-once**, so consumers see duplicates.

### Idempotency-Key pattern (Stripe-style)
```
POST /payments   Idempotency-Key: 5f1c…-uuid   (client generates one per logical operation)
```
```java
@Transactional
public PaymentResponse pay(String idemKey, PaymentRequest req) {
    Optional<IdempotencyRecord> existing = idemRepo.findById(idemKey);
    if (existing.isPresent()) {
        if (!existing.get().requestHash().equals(hash(req))) throw new IdempotencyKeyReuseException();  // 422
        return existing.get().response();                 // replay the stored response, no second charge
    }
    PaymentResponse res = gateway.charge(req);            // (often: insert the key FIRST with status=IN_PROGRESS)
    idemRepo.save(new IdempotencyRecord(idemKey, hash(req), res, Instant.now()));
    return res;
}
```
Key details:
- Store the key with a **UNIQUE constraint**. Two concurrent requests with the same key: one inserts, the other gets a constraint violation and waits, polls, or gets 409 "in progress".
- Store the **response** so retries get the identical answer. Expire keys after N hours (TTL).
- Scope keys per user/client. Reject the same key with a different payload.

### Idempotent consumers (Kafka/RabbitMQ)
Keep a `processed_messages(message_id UNIQUE)` table and insert it **in the same transaction** as the business change. A duplicate gets a constraint violation, so you skip it. Or make the operation naturally idempotent (`UPSERT`, "set status=PAID" instead of "add 100").

### Other idempotency tools
- Natural keys: `UNIQUE(order_id)` on payments.
- State machines: "PAID → PAID" is a no-op.
- Disable double-submit in the UI (helps, but **never** rely on the client alone).

---

## Interview Q&A
**Q1. Optimistic vs pessimistic locking?**
Detect conflicts with a version check (no locks, retry on failure; low contention) vs lock rows with `SELECT … FOR UPDATE` (blocking; high contention / critical updates).

**Q2. How does `@Version` work?**
Hibernate adds `AND version = ?` to UPDATEs and increments it. Zero affected rows throws `OptimisticLockException`.

**Q3. Two users book the last seat simultaneously. How do you prevent overbooking?**
A unique constraint on the seat, plus an atomic conditional update or a pessimistic lock in a short transaction. Return 409 to the loser.

**Q4. Why doesn't `synchronized` solve this?**
It only coordinates threads in one JVM. Multiple instances (and other writers) need DB-level or distributed coordination.

**Q5. What is idempotency? Which HTTP methods are idempotent?**
Same effect on repetition. GET, PUT, DELETE (and HEAD/OPTIONS) are; POST isn't.

**Q6. How do you make a payment API safe against retries?**
A client-generated Idempotency-Key stored with a unique constraint plus the saved response. Replay it on repeat; reject the same key with a different payload.

**Q7. How do you handle duplicate messages in Kafka consumers?**
At-least-once delivery means dedupe by message id (processed table in the same transaction) or naturally idempotent writes (upserts).

**Q8. How do you avoid deadlocks with pessimistic locks?**
A consistent lock ordering (e.g. by id), short transactions, lock timeouts, and retry on deadlock errors.

## Quick revision
- Lost update → DB-level control; JVM locks don't scale out.
- Optimistic `@Version` (retry or 409; ETag/If-Match) vs pessimistic `FOR UPDATE` (ordered, short transaction).
- Atomic `UPDATE … WHERE stock >= qty` + unique constraints are the strongest simple guards.
- Idempotency: GET/PUT/DELETE yes, POST no; Idempotency-Key + unique + stored response; idempotent consumers.
