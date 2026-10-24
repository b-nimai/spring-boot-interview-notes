# @Async and @Scheduled Tasks

> **Part of:** Spring gap topics (63–71) · **Prerequisite:** [59_Multithreading Basics](59_Multithreading%20Basics%20-%20Threads%2C%20Executors%20and%20CompletableFuture.md)
> 🟦 NestJS: `@nestjs/schedule` `@Cron()` ≈ `@Scheduled`; fire-and-forget promises / Bull queues ≈ `@Async` / message queues

## Why this topic matters
Sending emails without blocking the request, nightly jobs, and parallel calls are everyday backend tasks. Interviewers probe executor configuration, exception handling, self-invocation, and **scheduled jobs running on multiple instances**.

---

## 1. `@Async`
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean(name = "taskExecutor")                      // the default executor picked by @Async
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(4);
        ex.setMaxPoolSize(16);
        ex.setQueueCapacity(500);                     // bounded!
        ex.setThreadNamePrefix("async-");
        ex.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        ex.setWaitForTasksToCompleteOnShutdown(true); // graceful shutdown
        ex.setAwaitTerminationSeconds(30);
        ex.initialize();
        return ex;
    }
}

@Service
public class NotificationService {
    @Async                                             // runs on taskExecutor, caller returns immediately
    public void sendWelcomeEmail(String to) { mailClient.send(to); }

    @Async("reportExecutor")                           // a specific executor bean
    public CompletableFuture<Report> buildReport(Long id) {
        return CompletableFuture.completedFuture(heavyBuild(id));
    }
}
```
Boot also auto-configures a `ThreadPoolTaskExecutor` (`spring.task.execution.pool.core-size`, `max-size`, `queue-capacity`). With `spring.threads.virtual.enabled=true` (Boot 3.2+, Java 21), it uses virtual threads instead.

| Return type | Caller gets |
|---|---|
| `void` | Nothing; exceptions go to `AsyncUncaughtExceptionHandler` (logged by default) |
| `CompletableFuture<T>` | A future it can `join`/compose; exceptions complete it exceptionally |

### Exception handling for `void` methods
```java
@Configuration
public class AsyncErrors implements AsyncConfigurer {
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> log.error("Async {} failed", method.getName(), ex);
    }
}
```

### ⚠️ Pitfalls
- **Self-invocation** (`this.sendWelcomeEmail()`) runs synchronously (no proxy).
- `private` methods aren't proxied. Missing `@EnableAsync` means it runs synchronously.
- The **transaction doesn't propagate**: the async method runs in another thread, so it needs its own `@Transactional`. Entities passed in are **detached**; pass ids and reload.
- **SecurityContext / MDC (ThreadLocal) aren't propagated**. Use `DelegatingSecurityContextAsyncTaskExecutor` and a `TaskDecorator` to copy the MDC.
- **Async ≠ durable.** If the pod dies, queued tasks are lost. For must-not-lose work (emails after payment), use a message queue or the **outbox pattern** ([70](70_Kafka%20Basics%20with%20Spring.md)).
- Fire after commit: `@TransactionalEventListener(phase = AFTER_COMMIT)` + `@Async`, so the email isn't sent for a rolled-back order.

## 2. `@Scheduled`
```java
@Configuration
@EnableScheduling
public class SchedulingConfig { }

@Component
public class Jobs {
    @Scheduled(fixedRate = 60_000)              // every 60 s from the START of the previous run (can overlap if slow)
    public void heartbeat() { }

    @Scheduled(fixedDelay = 30_000, initialDelay = 10_000)   // 30 s after the previous run FINISHES
    public void syncInventory() { }

    @Scheduled(cron = "0 0 2 * * *", zone = "Asia/Kolkata")  // 02:00 every day
    public void nightlyReport() { }

    @Scheduled(fixedRateString = "${jobs.cleanup.rate:PT5M}")  // configurable (ISO-8601 duration)
    public void cleanup() { }
}
```
Spring cron = **6 fields**: `second minute hour day-of-month month day-of-week`
(`"0 */15 * * * MON-FRI"` = every 15 min on weekdays). Macros: `@daily`, `@hourly`.

- **Single scheduler thread by default**: one slow job delays all others. Configure `spring.task.scheduling.pool.size=5` or a `ThreadPoolTaskScheduler`.
- Methods must be `void` with no arguments.
- Exceptions are logged and the next run still happens. Handle errors and alert yourself.

### ⚠️ Multiple instances run the same job
With 3 pods, `@Scheduled` fires **3 times**. Options:
| Option | Idea |
|---|---|
| **ShedLock** | DB/Redis lock row per job (`@SchedulerLock(name = "nightlyReport", lockAtMostFor = "PT30M")`) |
| Leader election | Only the leader runs (Spring Integration / Kubernetes lease) |
| External scheduler | Kubernetes CronJob, Quartz cluster mode, AWS EventBridge calling an endpoint |
| Idempotent jobs | Safe to run twice anyway ([69](69_Optimistic%20and%20Pessimistic%20Locking%20and%20Idempotency.md)) |
**Quartz** offers persistent, clustered jobs, misfire handling, and dynamic triggers. **Spring Batch** handles large chunked ETL with restartability.

## 3. Choosing
| Need | Use |
|---|---|
| Non-critical side work off the request thread | `@Async` |
| Parallel calls, then combine | `CompletableFuture` + executor |
| Must not lose work, cross-service | Kafka/RabbitMQ (+ outbox) |
| Periodic job, single instance | `@Scheduled` |
| Periodic job, many instances | `@Scheduled` + ShedLock, or a K8s CronJob / Quartz cluster |
| Big batch processing | Spring Batch |

---

## Interview Q&A
**Q1. How does `@Async` work?**
`@EnableAsync` creates a proxy that submits the method call to a `TaskExecutor` and returns immediately (`void` or `CompletableFuture`).

**Q2. Why isn't my `@Async` method asynchronous?**
Self-invocation, a private method, missing `@EnableAsync`, or a non-bean object.

**Q3. How do you handle exceptions in `@Async void` methods?**
`AsyncUncaughtExceptionHandler` via `AsyncConfigurer`. With `CompletableFuture`, use `exceptionally`/`handle`.

**Q4. Does `@Transactional` carry over into an `@Async` method?**
No. It's a new thread, so declare its own transaction and reload entities by id.

**Q5. `fixedRate` vs `fixedDelay` vs `cron`?**
Fixed interval between starts / fixed gap after completion / calendar expression (6 fields in Spring).

**Q6. A `@Scheduled` job runs on all 3 instances. How do you fix it?**
ShedLock (distributed lock), leader election, an external scheduler (K8s CronJob/Quartz cluster), and an idempotent job design.

**Q7. Why configure the executor instead of using defaults?**
To bound threads and queues (avoid OOM and thread explosion), name threads, set rejection and shutdown behaviour, and isolate workloads.

**Q8. `@Async` vs a message queue?**
`@Async` is in-process and not durable (lost on crash). Queues are durable, retryable, and cross-service.

## Quick revision
- `@EnableAsync` + bounded `ThreadPoolTaskExecutor`; `void` or `CompletableFuture`.
- Proxy rules apply; no transaction/security/MDC propagation; not durable.
- `@EnableScheduling`; fixedRate / fixedDelay / cron (6 fields, zone); scheduler pool size.
- Multi-instance → ShedLock / leader / K8s CronJob; make jobs idempotent.
