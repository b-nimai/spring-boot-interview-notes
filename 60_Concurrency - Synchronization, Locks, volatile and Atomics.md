# Concurrency: Synchronization, Locks, volatile and Atomics

> **Part of:** Core Java (51–62) · **Prerequisite:** [59_Multithreading Basics](59_Multithreading%20Basics%20-%20Threads%2C%20Executors%20and%20CompletableFuture.md)

## Why this topic matters
Race conditions, `synchronized` vs `ReentrantLock`, `volatile`, atomics, deadlock, and producer-consumer are core product-company questions, and they come back in LLD (thread-safe singleton, rate limiter, parking lot).

---

## 1. The problems
### Race condition (lost update)
```java
class Counter { int count; void inc() { count++; } }   // count++ = read → add → write (NOT atomic)
// ✅ verified: 4 threads × 100,000 increments → plain int = 151,849 (lost updates!), AtomicInteger = 400,000
```
### Visibility
One thread's write may sit in a CPU cache or register, so another thread **never sees it**:
```java
boolean running = true;              // not volatile
while (running) { }                  // may loop forever even after another thread sets running = false
```
### Reordering
The compiler and CPU can reorder instructions. Without a **happens-before** relationship there's no ordering guarantee across threads.

**Java Memory Model (JMM) happens-before edges:** unlock → later lock of the same monitor; volatile write → later volatile read; `Thread.start()` → actions in the started thread; thread actions → `join()` returning; constructor final-field writes → readers (safe publication).

## 2. `synchronized` (intrinsic lock / monitor)
```java
public synchronized void inc() { count++; }          // locks `this`
public static synchronized void f() { }              // locks Counter.class
public void inc2() {
    synchronized (lock) { count++; }                 // block on a private lock object (preferred)
}
```
- **Mutual exclusion + visibility** (flushes on exit, refreshes on entry).
- **Reentrant**: the same thread can re-acquire a lock it holds.
- Waiting threads are **BLOCKED** and **can't be interrupted** while waiting; there's no timeout and no fairness.
- Lock on a `private final Object lock`, not `this` or a String literal (others could lock on it too).

### wait / notify: producer-consumer (classic)
```java
class BoundedBuffer<T> {
    private final Queue<T> q = new LinkedList<>(); private final int cap;
    BoundedBuffer(int cap) { this.cap = cap; }
    public synchronized void put(T x) throws InterruptedException {
        while (q.size() == cap) wait();      // ALWAYS while (spurious wakeups / re-check condition)
        q.add(x); notifyAll();
    }
    public synchronized T take() throws InterruptedException {
        while (q.isEmpty()) wait();
        T x = q.poll(); notifyAll(); return x;
    }
}
```
In practice use **`BlockingQueue`** (`ArrayBlockingQueue`, `LinkedBlockingQueue`): `put()` blocks when full, `take()` blocks when empty.

## 3. `volatile`
```java
private volatile boolean running = true;
```
- Guarantees **visibility** (reads see the latest write) and **ordering** (no reordering across it).
- Does **NOT** make compound actions atomic: `volatile int count; count++` still races.
- Use for flags, state published once, and double-checked locking.

### Thread-safe singleton (double-checked locking)
```java
public class Config {
    private static volatile Config instance;           // volatile is REQUIRED (prevents seeing a half-constructed object)
    private Config() {}
    public static Config getInstance() {
        if (instance == null) {                        // 1st check (no lock)
            synchronized (Config.class) {
                if (instance == null) instance = new Config();   // 2nd check
            }
        }
        return instance;
    }
}
// Simpler: holder idiom (lazy + thread-safe via class loading) or an enum singleton
class Holder { private static class H { static final Config I = new Config(); } static Config get() { return H.I; } }
enum Registry { INSTANCE; }
```

## 4. Atomics (lock-free CAS)
```java
AtomicInteger count = new AtomicInteger();
count.incrementAndGet();                         // atomic ++
count.compareAndSet(5, 6);                       // CAS: set to 6 only if currently 5
count.updateAndGet(x -> Math.max(x, newVal));
AtomicReference<State> state = new AtomicReference<>(INIT);
LongAdder hits = new LongAdder(); hits.increment();   // better than AtomicLong under heavy contention (striped cells)
```
**CAS (compare-and-swap)**: a CPU instruction that atomically does "if value == expected, set new". On failure it retries in a loop. Lock-free and non-blocking.
**ABA problem**: the value changes A→B→A and CAS wrongly succeeds. Fix with `AtomicStampedReference` (version stamp).

## 5. `java.util.concurrent.locks`
```java
private final ReentrantLock lock = new ReentrantLock(true);   // optional fairness
public void transfer() {
    if (lock.tryLock(1, TimeUnit.SECONDS)) {       // timeout → avoids deadlock
        try { /* critical section */ }
        finally { lock.unlock(); }                  // ALWAYS unlock in finally
    }
}
```
| `synchronized` | `ReentrantLock` |
|---|---|
| Implicit lock/unlock (block scoped) | Explicit; must `unlock()` in `finally` |
| No timeout; not interruptible while waiting | `tryLock()`, `tryLock(timeout)`, `lockInterruptibly()` |
| No fairness | Optional fair ordering |
| One wait-set (`wait/notify`) | Multiple `Condition`s (`await/signal`), e.g. notFull/notEmpty |
| Simpler, JVM-optimized | More control; use when you need these features |
- **`ReadWriteLock`** (`ReentrantReadWriteLock`): many concurrent readers or one writer, for read-heavy caches.
- **`StampedLock`** (8): optimistic reads; not reentrant.

## 6. Deadlock, livelock, starvation
```java
// Thread 1: lock(A) → lock(B)       Thread 2: lock(B) → lock(A)   → DEADLOCK
```
Four conditions (all required): **mutual exclusion, hold-and-wait, no preemption, circular wait**.
Prevention:
- **Global lock ordering** (e.g. always lock the lower account id first in transfers).
- `tryLock` with a timeout plus back-off.
- Hold locks briefly; don't call alien code while holding a lock.
- Detect with a thread dump (`jstack <pid>` shows "Found one Java-level deadlock").
**Livelock**: threads keep reacting to each other and make no progress. **Starvation**: a thread never gets the CPU or lock (unfair locks, priorities).

## 7. Synchronizers
| Class | Use |
|---|---|
| `CountDownLatch(n)` | Wait until n events happen (`countDown()` / `await()`); **one-shot** |
| `CyclicBarrier(n)` | n threads wait for each other at a point; **reusable** |
| `Semaphore(permits)` | Limit concurrent access (connection limits, rate limiting) |
| `Phaser` | Flexible multi-phase barrier |
| `Exchanger` | Two threads swap objects |
```java
Semaphore permits = new Semaphore(3);
permits.acquire(); try { callExternalApi(); } finally { permits.release(); }   // max 3 concurrent calls
```

## 8. Concurrent collections
| Collection | Notes |
|---|---|
| `ConcurrentHashMap` | Fine-grained, atomic `compute`/`merge` ([55](55_HashMap%20Internals.md)) |
| `CopyOnWriteArrayList`/`Set` | Copies on every write; great for read-mostly listener lists |
| `BlockingQueue` (`ArrayBlockingQueue`, `LinkedBlockingQueue`, `PriorityBlockingQueue`, `DelayQueue`, `SynchronousQueue`) | Producer-consumer, thread pools |
| `ConcurrentLinkedQueue` | Lock-free queue |
| `ConcurrentSkipListMap`/`Set` | Sorted concurrent map/set |

## 9. Thread-safety strategies (in order of preference)
1. **No shared state** (stateless services, local variables, thread confinement).
2. **Immutability** ([53](53_equals%2C%20hashCode%2C%20Immutability%20and%20Records.md)).
3. **Concurrent collections / atomics**.
4. **Locks** (`synchronized`/`ReentrantLock`), with minimal scope.
5. **Database-level** locking for cross-instance consistency ([43](43_Transaction%20Propagation%20and%20Isolation.md)). JVM locks don't work across multiple app instances. Use DB locks, `@Version`, or Redis distributed locks.

---

## Interview Q&A
**Q1. What is a race condition? How do you fix `count++`?**
Unsynchronized read-modify-write by multiple threads. Fix with `AtomicInteger`, `synchronized`, or a lock.

**Q2. `volatile` vs `synchronized`?**
`volatile` gives visibility and ordering only, with no atomicity for compound ops. `synchronized` gives mutual exclusion, visibility, and atomicity of the block.

**Q3. Why is `volatile` needed in double-checked locking?**
Without it another thread may see a non-null reference to a partially constructed object (reordering of allocation, assignment, and init).

**Q4. `synchronized` vs `ReentrantLock`?**
Implicit and simple vs explicit with tryLock/timeout, interruptible waits, fairness, and multiple conditions.

**Q5. What is CAS? What is the ABA problem?**
Atomic compare-and-swap (lock-free retries). ABA is a value cycling back to the expected one; fix with a stamped/versioned reference.

**Q6. How does deadlock happen and how do you prevent it?**
Circular wait on locks. Prevent with consistent lock ordering, timeouts (`tryLock`), and minimal lock scope; diagnose with `jstack`.

**Q7. `CountDownLatch` vs `CyclicBarrier`?**
A one-shot wait for N events (counter reaches 0) vs N threads waiting for each other at a reusable barrier.

**Q8. Why must `wait()` be called in a loop?**
Spurious wakeups, and the condition may have changed before the thread re-acquired the lock.

**Q9. Is `ConcurrentHashMap` enough to make check-then-act safe?**
Only with its atomic methods (`putIfAbsent`, `computeIfAbsent`, `merge`). Separate `get` + `put` still races.

**Q10. How do you make an operation thread-safe across 3 instances of a Spring Boot service?**
JVM locks don't help. Use DB row locks / optimistic `@Version`, unique constraints, or a distributed lock (Redis/Zookeeper).

## Quick revision
- Problems: atomicity (race), visibility (caches), ordering (reordering); happens-before fixes them.
- `synchronized` = exclusion + visibility, reentrant; wait/notify in a while loop (or BlockingQueue).
- `volatile` = visibility only; DCL singleton needs it (or holder/enum).
- Atomics = CAS (lock-free); LongAdder for hot counters.
- ReentrantLock: tryLock/fair/conditions, unlock in finally; ReadWriteLock for read-heavy.
- Deadlock = circular wait → lock ordering + timeouts. Cross-instance → DB/distributed locks.
