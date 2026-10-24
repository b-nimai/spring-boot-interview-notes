# Multithreading Basics: Threads, Executors and CompletableFuture

> **Part of:** Core Java (51–62) · **Next:** [60_Synchronization, Locks, volatile and Atomics](60_Concurrency%20-%20Synchronization%2C%20Locks%2C%20volatile%20and%20Atomics.md)
> 🟦 **Biggest shift from Node:** Node runs your JS on one thread (the event loop). Java runs **many threads in parallel on shared memory**, so shared mutable state needs coordination.

## Why this topic matters
Concurrency is a must at product companies: thread lifecycle, Runnable vs Callable, thread pools, `CompletableFuture`, and how Spring's `@Async` and Tomcat threads relate.

---

## 1. Process vs thread
| Process | Thread |
|---|---|
| Own memory space | Shares the process heap; own **stack** + program counter |
| Heavy to create; IPC to communicate | Lighter; communicates via shared memory (needs synchronization) |
A Spring MVC app = one JVM process with ~200 Tomcat worker threads, each request on its own thread ([21](21_Servlets%2C%20Tomcat%20and%20Servlet%20Lifecycle.md)).
**Concurrency** = dealing with many tasks (interleaving). **Parallelism** = executing simultaneously on multiple cores.

## 2. Creating threads
```java
// 1) Runnable (preferred over extending Thread: composition, the class can extend something else)
Thread t1 = new Thread(() -> System.out.println("in " + Thread.currentThread().getName()));
t1.start();          // ✅ new thread → run() executes there
// t1.run();         // ❌ just a normal method call on the CURRENT thread

// 2) extend Thread (rarely)
class Worker extends Thread { public void run() { ... } }

// 3) Callable → returns a value / throws checked exceptions (used with executors)
Callable<Integer> task = () -> 42;

// Java 21: virtual threads
Thread.ofVirtual().start(() -> handle());
```
| Runnable | Callable |
|---|---|
| `void run()` | `V call() throws Exception` |
| No result | Result via `Future<V>` |
Calling `start()` twice → `IllegalThreadStateException`.

## 3. Thread lifecycle
```
NEW ──start()──► RUNNABLE ⇄ (running on CPU)
                   │  ├─ synchronized lock not available ──► BLOCKED
                   │  ├─ wait() / join() / park() ─────────► WAITING
                   │  └─ sleep(ms) / wait(ms) / join(ms) ──► TIMED_WAITING
                   ▼
              TERMINATED
```
| Method | Effect |
|---|---|
| `Thread.sleep(ms)` | Pause; **keeps locks**; throws `InterruptedException` |
| `t.join()` | Wait for t to finish |
| `Thread.yield()` | Hint to give up the CPU |
| `t.interrupt()` | Cooperative cancellation: sets a flag / wakes sleep/wait with `InterruptedException` |
| `t.setDaemon(true)` | Background thread; the JVM exits without waiting for it |
| `obj.wait()` / `notify()` | Must hold `obj`'s monitor; `wait` **releases** the lock ([60](60_Concurrency%20-%20Synchronization%2C%20Locks%2C%20volatile%20and%20Atomics.md)) |
`stop()`/`suspend()` are deprecated (unsafe). Use interruption plus a check of `Thread.currentThread().isInterrupted()`.

## 4. Executor framework (thread pools)
Creating a thread per task is expensive and unbounded. Reuse a pool:
```java
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<Integer> f = pool.submit(() -> compute());   // Callable
Integer result = f.get(2, TimeUnit.SECONDS);         // blocks; timeout
pool.execute(() -> log());                           // Runnable, fire-and-forget
List<Future<Integer>> all = pool.invokeAll(tasks);
pool.shutdown();                                     // stop accepting; finish queued tasks
pool.awaitTermination(30, TimeUnit.SECONDS);
// pool.shutdownNow() → interrupts running tasks; returns queued ones
// Java 19+: try (var ex = Executors.newVirtualThreadPerTaskExecutor()) { ... }  (auto-close)
```
| Factory | Behaviour | Risk |
|---|---|---|
| `newFixedThreadPool(n)` | n threads, **unbounded** queue | Queue can grow → OOM |
| `newCachedThreadPool()` | Grows without limit, reuses idle threads | Thread explosion |
| `newSingleThreadExecutor()` | Sequential execution | — |
| `newScheduledThreadPool(n)` | Delayed/periodic (`scheduleAtFixedRate`) | — |
| `newWorkStealingPool()` | ForkJoinPool | — |
| `newVirtualThreadPerTaskExecutor()` (21) | One cheap virtual thread per task | Don't pool virtual threads |

### ThreadPoolExecutor (what production code configures)
```java
ThreadPoolExecutor ex = new ThreadPoolExecutor(
    4,                              // corePoolSize
    8,                              // maximumPoolSize
    60, TimeUnit.SECONDS,           // keepAlive for idle non-core threads
    new ArrayBlockingQueue<>(100),  // BOUNDED queue
    new ThreadPoolExecutor.CallerRunsPolicy());   // rejection policy
```
Task submission order: **core threads → queue → extra threads up to max → reject**. (With an unbounded queue, max is never reached.)
Rejection policies: `AbortPolicy` (default, throws), `CallerRunsPolicy` (back-pressure), `DiscardPolicy`, `DiscardOldestPolicy`.
Sizing heuristics: CPU-bound ≈ `#cores`; I/O-bound ≈ `#cores × (1 + wait/compute)`.

## 5. Future vs CompletableFuture
`Future` is limited: `get()` blocks, no chaining or combining, and no manual completion.
`CompletableFuture` (Java 8) is Java's **Promise**:
```java
CompletableFuture<User> userF = CompletableFuture.supplyAsync(() -> userClient.get(id), ioPool);
CompletableFuture<List<Order>> ordersF = CompletableFuture.supplyAsync(() -> orderClient.list(id), ioPool);

CompletableFuture<Dashboard> dash = userF.thenCombine(ordersF, Dashboard::new)   // combine two
        .orTimeout(2, TimeUnit.SECONDS)                                          // Java 9
        .exceptionally(ex -> Dashboard.empty());                                 // fallback

CompletableFuture.allOf(userF, ordersF).join();       // like Promise.all (returns Void → read each with join())
CompletableFuture.anyOf(f1, f2);                       // like Promise.race
```
| Method | Like (JS) | Meaning |
|---|---|---|
| `supplyAsync(sup, ex)` / `runAsync` | `new Promise` | Start async work (default pool: ForkJoin common pool; **pass your own executor for I/O**) |
| `thenApply(fn)` | `.then(x => y)` | Transform the result |
| `thenCompose(fn)` | `.then(x => promise)` | Chain a dependent async call (flatMap) |
| `thenCombine(other, fn)` | `Promise.all` of 2 | Combine two independent results |
| `thenAccept` / `thenRun` | `.then(x => {…})` | Consume / run after |
| `exceptionally(fn)` | `.catch` | Recover |
| `handle((r, ex) → …)` | `.then(ok, err)` | Both paths |
| `whenComplete` | `.finally` | Side effect, result unchanged |
| `join()` / `get()` | `await` | Block for the result (`join` throws unchecked `CompletionException`) |
`thenApply` vs `thenApplyAsync`: the `*Async` variants run the callback on a pool thread rather than the completing thread.

## 6. Spring connections
- **`@Async`** (+ `@EnableAsync`): the method runs on a `TaskExecutor`. Return `void` or `CompletableFuture<T>`. It's a proxy, so **self-invocation doesn't work**. Configure a bounded `ThreadPoolTaskExecutor`. Boot 3.2+ can use virtual threads with `spring.threads.virtual.enabled=true`.
- **`@Scheduled(fixedRate=…, cron=…)`** + `@EnableScheduling`: periodic jobs (single scheduler thread by default).
- `@Transactional` and `SecurityContext` are **ThreadLocal-bound**, so they don't propagate to new threads automatically (use `DelegatingSecurityContextExecutor`, and a new transaction per thread).
- Singleton beans are shared across request threads, so **keep them stateless** ([11](11_Bean%20Scopes.md)).

## 7. ThreadLocal
```java
private static final ThreadLocal<String> requestId = new ThreadLocal<>();
requestId.set(UUID.randomUUID().toString());
try { ... requestId.get() ... } finally { requestId.remove(); }   // ⚠️ always remove (pooled threads reuse values → leaks)
```
Used by `SecurityContextHolder`, transaction synchronization, and MDC logging.

---

## Interview Q&A
**Q1. `start()` vs `run()`?**
`start()` creates a new thread that calls `run()`. Calling `run()` directly executes on the current thread.

**Q2. Runnable vs Callable?**
No result and no checked exceptions vs returns a value via `Future` and can throw.

**Q3. Thread states?**
NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED.

**Q4. `sleep()` vs `wait()`?**
`sleep` is a Thread static that keeps locks. `wait` is an Object method, requires the monitor, releases it, and is woken by `notify` or timeout.

**Q5. Why use thread pools? What's dangerous about `newFixedThreadPool`/`newCachedThreadPool`?**
Reuse threads and bound resources. Fixed has an unbounded queue (OOM); cached has unbounded threads. Prefer a configured `ThreadPoolExecutor`.

**Q6. Explain the ThreadPoolExecutor parameters and task flow.**
core, max, keepAlive, queue, rejection handler. Tasks fill core threads → the queue → threads up to max → rejection.

**Q7. Future vs CompletableFuture?**
Blocking `get` only vs non-blocking composition (`thenApply`/`thenCompose`/`thenCombine`/`allOf`), exception handling, and manual completion.

**Q8. `thenApply` vs `thenCompose`?**
Sync transformation (map) vs chaining a function that returns another `CompletableFuture` (flatMap).

**Q9. How do you stop a thread?**
Cooperatively: `interrupt()` plus checking the interrupted flag / handling `InterruptedException`, or a volatile flag. Never `stop()`.

**Q10. Why might `@Async` not run asynchronously?**
Missing `@EnableAsync`, self-invocation (no proxy), a private method, or calling it on a non-bean.

## Quick revision
- Threads share the heap and have their own stack; Runnable/Callable; `start()` not `run()`.
- States: NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED; interruption is cooperative.
- Use executors; configure bounded `ThreadPoolExecutor` (core → queue → max → reject); always shutdown.
- `CompletableFuture` = Promise (`thenApply`/`thenCompose`/`thenCombine`/`allOf`/`exceptionally`); custom executor for I/O.
- Spring `@Async`/`@Scheduled` are proxy-based; ThreadLocal context doesn't cross threads.
