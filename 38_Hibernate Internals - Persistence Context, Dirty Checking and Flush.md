# Hibernate Internals: Persistence Context, Dirty Checking and Flush

> **Source:** Lecture 29 (`Notes_Of_Aditya/Lecture 29/code/hibernateInternalsDemo`)
> **Prerequisite:** [37_Hibernate and JPA Entity Mapping](37_Hibernate%20and%20JPA%20Entity%20Mapping.md) · **Related:** [34_AOP Proxy Internals](34_Custom%20Annotations%20and%20AOP%20Proxy%20Internals.md)

## Why this topic matters
This is the most interview-heavy Hibernate topic: entity states, `persist` vs `merge`, dirty checking, flush vs commit, the first-level cache, and why `@Transactional` matters. It explains "why did my update happen without `save()`?" and "why didn't my change save?".

---

## 1. Core components
| Component | Role | Lifetime |
|---|---|---|
| `EntityManagerFactory` (Hibernate: `SessionFactory`) | Built once from mapping metadata; creates EntityManagers | Whole app (expensive, thread-safe) |
| `EntityManager` (Hibernate: `Session`) | API to work with entities | One unit of work / transaction (**not thread-safe**) |
| **Persistence Context** | In-memory workspace of **managed** entities | Usually one transaction |
| DB transaction | All-or-nothing unit | One business operation |
```
EntityManagerFactory → creates → EntityManager → works with → Persistence Context → (flush) → Database
```

## 2. Persistence context
```
Persistence Context
 ├─ (Student, 1) → Student@A
 ├─ (Student, 2) → Student@B
 └─ snapshots of loaded state
```
Provides: identity map, entity state tracking, dirty checking, **first-level cache**, transactional write-behind.

### `@PersistenceContext` in a singleton repository
The repository is a singleton, but `EntityManager` isn't thread-safe. Spring injects a **shared proxy** that routes each call to the EntityManager bound to the **current transaction**:
```
Request A → Tx A → EntityManager A → Persistence Context A
Request B → Tx B → EntityManager B → Persistence Context B
```

## 3. Transactions and `@Transactional`
Bank transfer: debit A + credit B must both succeed or both roll back. That's **atomicity**.
```java
@Transactional          // Spring AOP proxy: begin → method → flush → commit   (or rollback)
public void updateStudent(Long id) { ... }
```
- Rolls back on **unchecked** exceptions (`RuntimeException`, `Error`) by default. Checked exceptions **commit** unless `rollbackFor = Exception.class`.
- Implemented via an AOP proxy, so **self-invocation and private methods bypass it**.
- **Put it in the service layer**: one business use case (enroll + pay) = one transaction. Repository-level transactions would split it.
- `@Transactional(readOnly = true)` for reads (Spring's annotation): hints to skip dirty checking/flush.

| Operation | Needs a transaction? |
|---|---|
| `find()` | Not strictly (unless locking) |
| `persist`, `merge`, `remove`, `flush`, `refresh` | Yes |

## 4. Entity lifecycle states
```
new Student()  ──persist()──►  MANAGED  ──remove()──►  REMOVED ──flush/commit──► row deleted
   TRANSIENT                    ▲   │
                       find()───┘   │ detach() / clear() / tx ends
                                    ▼
                                DETACHED ──merge()──► returns a (different) MANAGED copy
```
| State | In persistence context? | Row in DB? | Changes tracked? |
|---|---|---|---|
| **Transient** | ❌ | No (not yet) | ❌ |
| **Managed** | ✅ | Yes or pending insert | ✅ |
| **Detached** | ❌ (was) | Yes | ❌ |
| **Removed** | ✅ (scheduled) | Deleted at flush | — |

- `@Entity` doesn't make `new` objects managed. They're transient until `persist`.
- **Managed ≠ already written**: INSERT may wait until flush (with `IDENTITY`, Hibernate inserts immediately on `persist` to get the id).
- `detach()` ≠ delete. `remove()` deletes.
- **`merge()` returns a NEW managed instance**. The argument stays detached:
  ```java
  Student managed = entityManager.merge(detached);   // keep using `managed`, not `detached`
  ```
- `entityManager.contains(s)` tells you whether `s` is managed.

## 5. Identity map
```java
Student a = em.find(Student.class, 1L);
Student b = em.find(Student.class, 1L);
a == b   // true: one managed instance per (type, id) per persistence context
```

## 6. Dirty checking
```java
@Transactional
public void updateStudent(Long id) {
    Student s = em.find(Student.class, id);   // managed + snapshot taken
    s.setName("Rohan");                       // only Java memory changes
}                                             // flush: compare with snapshot → UPDATE students SET name=? WHERE id=?
```
- "Dirty" = current state ≠ loaded snapshot (not "invalid").
- Setters don't run SQL. Detection happens at **flush**.
- Setting the same value means no change and no UPDATE.
- **Detached entities aren't dirty-checked**, so changes made after `detach()` or outside the transaction are lost unless you `merge`.

## 7. Flush vs commit
| `flush()` | `commit()` |
|---|---|
| Sends pending SQL to the DB | Makes the transaction durable and ends it |
| Still inside the transaction, can still roll back | Final |

Automatic flush (`FlushModeType.AUTO`): **before commit** and **before queries** whose results could be affected by pending changes.
Manual `flush()`: surface constraint violations early, sync before native SQL, or in batch loops.

### Lecture's flush experiment
```java
@Transactional
public void updateStudent(Student req, Long id) {
    Student s = studentRepository.findById(id);
    s.setName(req.getName());   studentRepository.flush();   // UPDATE #1 sent
    s.setAge(req.getAge());     studentRepository.flush();   // UPDATE #2 sent
    s.setEmail(req.getEmail()); studentRepository.flush();   // UPDATE #3 sent
}                                                            // commit makes all 3 durable together
```
`show-sql` shows **three** UPDATE statements instead of one. Explicit flushes defeat write-behind batching. If an exception occurs after the second flush, the whole transaction **still rolls back**, because flushed ≠ committed.

## 8. Transactional write-behind
```
setX ─┐
setY ─┼──► persistence context ──flush──► ordered/batched SQL
setZ ─┘
```
Write-behind lets Hibernate skip redundant work, order statements (FK-safe), and batch them.

## 9. Batch inserts: `flush()` + `clear()`
```java
public void saveAll(List<Student> students) {          // lecture code
    int counter = 0;
    for (Student s : students) {
        entityManager.persist(s);
        if (++counter % 100 == 0) {
            entityManager.flush();     // send pending INSERTs
            entityManager.clear();     // detach all → free memory
        }
    }
}
```
| `flush()` | `clear()` |
|---|---|
| Executes SQL; entities stay managed | Detaches **everything**; empties the context |

> ⚠️ This controls **memory**, but real JDBC batching also needs `spring.jpa.properties.hibernate.jdbc.batch_size=100` (+ `order_inserts=true`). And **`GenerationType.IDENTITY` disables insert batching** in Hibernate (it must insert each row to get its id). Use `SEQUENCE` (PostgreSQL/Oracle) or other strategies for true batch inserts. The lecture project uses IDENTITY on MySQL, so each row is still a separate INSERT.

## 10. First-level cache
- Key = (entity type, id). **On by default, can't be disabled, scoped to one persistence context** (≈ one transaction).
```java
@Transactional void a() { em.find(Student.class, 1L); em.find(Student.class, 1L); }  // 1 SELECT
@Transactional void b() { em.find(Student.class, 1L); }                               // new tx → SELECT again
```
- `clear()` or `detach()` → the next `find` hits the DB again and returns a **different** object (`first != second`).
- JPQL/Criteria queries **still run SQL**, though Hibernate returns the already-managed instances for identity.
- **Second-level cache** (optional, e.g. Ehcache/Caffeine) is shared across sessions. Different thing.

`refresh(entity)`: reloads state from the DB and **discards unsynchronized in-memory changes**.

## 11. EntityManager cheat sheet
| Method | Effect |
|---|---|
| `persist(e)` | Transient → managed; INSERT scheduled |
| `find(T.class, id)` | Managed entity (cache or SELECT) or `null` |
| `getReference(T.class, id)` | Lazy proxy, no SELECT until accessed |
| `merge(e)` | Copies state into a managed instance and **returns it** |
| `remove(e)` | Managed → removed; DELETE scheduled |
| `contains(e)` | Is it managed? |
| `detach(e)` / `clear()` | Detach one / all |
| `flush()` | Sync pending changes to the DB |
| `refresh(e)` | Overwrite with DB state |

## 12. Misconceptions
❌ `@Entity` objects are automatically managed · ❌ a setter runs UPDATE immediately · ❌ managed = already in the DB · ❌ flush = commit · ❌ `detach` deletes · ❌ `merge` reattaches the same object · ❌ the first-level cache is app-wide · ❌ a singleton repository shares one EntityManager.

---

## Interview Q&A
**Q1. What is the persistence context?**
Hibernate's per-transaction workspace of managed entities. It provides the identity map, first-level cache, dirty checking, and write-behind.

**Q2. Explain the entity lifecycle states.**
Transient (new, untracked) → managed (tracked) → detached (no longer tracked) or removed (scheduled for delete).

**Q3. What is dirty checking?**
At flush, Hibernate compares managed entities with their snapshots and issues UPDATEs for changed fields. No explicit save is needed.

**Q4. `flush()` vs `commit()`?**
Flush sends SQL inside the transaction. Commit makes it permanent. Flushed changes can still be rolled back.

**Q5. `persist()` vs `merge()`?**
`persist` makes the given new instance managed (returns void). `merge` copies a detached or new object's state into a managed instance and returns that instance.

**Q6. What is the first-level cache? Can you disable it?**
The persistence-context cache keyed by (type, id), scoped to a session/transaction. It's always on.

**Q7. Why must `EntityManager` not be shared across threads? How does Spring handle it?**
It isn't thread-safe. Spring injects a transaction-aware shared proxy.

**Q8. Why might `@Transactional` not roll back?**
A checked exception (without `rollbackFor`), self-invocation or a private method (no proxy), or the exception was caught and swallowed.

**Q9. How do you insert 100,000 rows efficiently?**
Configure `hibernate.jdbc.batch_size`, avoid IDENTITY (use SEQUENCE), and `flush()` + `clear()` every N rows. Consider JdbcTemplate `batchUpdate` instead.

**Q10. Why put `@Transactional` on the service, not the repository?**
The service method is the business unit of work, and all its repository calls must commit or roll back together.

## Quick revision
- EMF (app) → EM (tx) → persistence context (managed entities).
- Transient → managed → detached / removed. `merge` returns a new managed copy.
- Dirty checking at flush, so no `update()` is needed.
- Flush ≠ commit. Auto-flush happens before commit and relevant queries.
- First-level cache is per transaction; `flush`+`clear` for batches; IDENTITY blocks insert batching.
