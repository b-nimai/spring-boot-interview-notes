# JVM Memory, Class Loading and Garbage Collection

> **Part of:** Core Java (51–62)

## Why this topic matters
"Explain JVM memory areas", "how does GC work", "what causes memory leaks in Java", and "how would you debug an OutOfMemoryError" are standard at product companies, especially for backend roles where you'll tune services in containers.

---

## 1. JVM architecture
```
.class files → ClassLoader subsystem → Runtime Data Areas → Execution Engine (Interpreter + JIT + GC)
                                                         ↘ JNI → native libs
```

## 2. Runtime data areas
| Area | Shared? | Holds | Error |
|---|---|---|---|
| **Heap** | All threads | All objects and arrays (incl. the String pool since Java 7) | `OutOfMemoryError: Java heap space` |
| **Metaspace** (Java 8+, replaced PermGen) | All threads | Class metadata, method bytecode, (static vars live on the heap with the Class object) | `OutOfMemoryError: Metaspace` |
| **JVM stack** | Per thread | Frames: local variables, operand stack, references | `StackOverflowError` |
| **PC register** | Per thread | Current instruction address | — |
| **Native method stack** | Per thread | Native (JNI) calls | — |
| Code cache | Shared | JIT-compiled native code | — |
Metaspace uses **native memory** (auto-grows; cap with `-XX:MaxMetaspaceSize`). PermGen was fixed-size and caused frequent OOMs.

```java
void m() {
    int x = 5;                  // stack (primitive local)
    User u = new User("A");     // reference u on the stack → User object on the HEAP
}
```
**Escape analysis**: the JIT may allocate non-escaping objects on the stack (scalar replacement).

## 3. Class loading
**Load → Link (verify → prepare → resolve) → Initialize** (static initializers run, on first active use).
| Loader | Loads |
|---|---|
| Bootstrap (native) | Core JDK (`java.base`: `java.lang.*`) |
| Platform (formerly Extension) | Other JDK modules |
| Application (System) | Your classpath (app + dependencies) |
**Parent delegation model**: a loader first asks its **parent**. This prevents a fake `java.lang.String` from replacing the real one and avoids duplicate loading.
A class is identified by **(fully-qualified name + its classloader)**. The same class loaded by two loaders gives `ClassCastException: X cannot be cast to X` (seen with app servers and Spring DevTools restarts).

## 4. Garbage collection basics
The GC frees objects **not reachable from GC roots**.
**GC roots:** local variables/references on thread stacks, static fields, active threads, JNI references, monitors.
Reachability-based, so **circular references are collected fine** (unlike naive reference counting).

### Generational hypothesis
Most objects die young, so the heap is split:
```
Heap
 ├─ Young Generation
 │    ├─ Eden            (new objects allocated here)
 │    ├─ Survivor S0
 │    └─ Survivor S1
 └─ Old (Tenured) Generation   (long-lived objects promoted after N survivals, ~15 by default)
```
- **Minor GC**: collects the Young gen. Live objects in Eden + one survivor are copied to the other survivor (copying = fast, no fragmentation); ages increment; old enough → promoted to Old. Frequent and fast.
- **Major/Full GC**: collects Old (+ everything). Slower, longer pauses.
- **Stop-The-World (STW)**: application threads paused during (parts of) a GC.

### Algorithms
- **Mark** (trace live objects from roots) → **Sweep** (free dead ones, leaving fragmentation) → **Compact** (move live objects together).
- **Copying** (young gen): copy live objects to a new space.

### Collectors
| Collector | Idea | Use |
|---|---|---|
| Serial | Single-threaded STW | Tiny heaps / containers with 1 CPU |
| Parallel | Multi-threaded STW, max **throughput** | Batch jobs |
| **G1** (default since Java 9) | Heap split into **regions**; collects the regions with the most garbage first; pause-time target (`-XX:MaxGCPauseMillis=200`) | General-purpose servers |
| **ZGC** (prod Java 15; generational in 21) | Concurrent, colored pointers, **sub-millisecond pauses**, huge heaps | Latency-sensitive services |
| Shenandoah | Concurrent compaction, low pause | Low latency |
(CMS was removed in Java 14.)

## 5. Reference types
| Type | Collected when | Use |
|---|---|---|
| Strong (normal) | Never while reachable | Everything |
| `SoftReference` | Memory is low | Memory-sensitive caches |
| `WeakReference` | Next GC if only weakly reachable | `WeakHashMap`, canonical maps, listeners |
| `PhantomReference` | After finalization; used with a `ReferenceQueue` | Cleanup hooks (`Cleaner`) |

## 6. Memory leaks in Java (yes, they happen)
Objects that are still **reachable but no longer needed**:
- Ever-growing **static collections/caches** with no eviction (use bounded caches like Caffeine, with size/TTL).
- **ThreadLocal** values not removed in pooled threads.
- Unclosed resources (streams, connections), so use try-with-resources.
- Listeners/callbacks registered and never unregistered.
- Inner (non-static) classes holding outer instances; lambdas capturing large objects.
- **Mutable HashMap keys** (entries unreachable by lookup but still referenced, see [55](55_HashMap%20Internals.md)).
- Classloader leaks (redeploys in app servers) → Metaspace OOM.
- JPA: a huge persistence context in batch jobs without `clear()` ([38](38_Hibernate%20Internals%20-%20Persistence%20Context%2C%20Dirty%20Checking%20and%20Flush.md)).

## 7. Tuning and troubleshooting
```bash
java -Xms512m -Xmx512m                    # initial / max heap (set equal in servers to avoid resizing)
     -Xss512k                             # thread stack size
     -XX:+UseG1GC -XX:MaxGCPauseMillis=200
     -XX:MaxRAMPercentage=75              # in containers: size heap relative to the cgroup memory limit
     -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp
     -Xlog:gc*:file=gc.log                # GC logging (Java 9+ unified logging)
```
| Tool | Use |
|---|---|
| `jps` | List JVMs |
| `jstack <pid>` | Thread dump (deadlocks, stuck threads) |
| `jmap -dump:live,format=b,file=heap.hprof <pid>` / `jcmd <pid> GC.heap_dump` | Heap dump |
| `jstat -gc <pid> 1s` | GC stats live |
| `jcmd` | Swiss-army knife (`Thread.print`, `VM.flags`, `GC.class_histogram`) |
| VisualVM, JDK Mission Control + **JFR**, Eclipse **MAT** | Profiling, leak analysis (dominator tree) |
| Spring Boot **Actuator** + Micrometer | `jvm.memory.used`, `jvm.gc.pause` metrics in production |
**Debugging an OOM:** get a heap dump → open it in MAT → find the biggest retained objects (dominator tree) → follow the path to the GC root → fix the leak (or raise the heap if it's legitimate). For high CPU, take several thread dumps, and check GC logs for GC thrashing.

## 8. JIT and performance notes
- Code is interpreted first, then hot methods are compiled by C1 (fast) and C2 (optimized): **tiered compilation**. Warm-up matters for benchmarks; use **JMH**.
- `System.gc()` is only a hint; avoid it.
- `finalize()` is deprecated (unpredictable). Use try-with-resources or `Cleaner`.
- GraalVM native image (Spring Boot 3 AOT) gives fast startup and low memory, but closed-world restrictions.

---

## Interview Q&A
**Q1. Explain the JVM memory areas.**
Heap (objects, shared), Metaspace (class metadata, native memory), per-thread stack (frames/locals), PC register, native stack, code cache.

**Q2. Stack vs heap?**
Per-thread frames for locals/references, auto-freed vs shared object storage managed by the GC. StackOverflowError vs OutOfMemoryError.

**Q3. PermGen vs Metaspace?**
Java 8 replaced the fixed-size PermGen (in heap space) with Metaspace in native memory, which auto-grows and is cappable.

**Q4. How does GC decide what to collect?**
Reachability from GC roots (stack references, statics, threads, JNI). Unreachable objects, even circular ones, are garbage.

**Q5. Explain generational GC.**
Young (Eden + 2 survivors, frequent fast copying minor GCs) and Old (promoted long-lived objects, less frequent major GCs), based on "most objects die young".

**Q6. Which GC would you choose?**
G1 (default) for general services, ZGC/Shenandoah for low latency or huge heaps, Parallel for throughput-oriented batch jobs.

**Q7. Can Java have memory leaks? Examples?**
Yes, through unintentionally retained references: static caches, ThreadLocals, listeners, unclosed resources, mutable map keys.

**Q8. Explain the parent delegation model.**
A classloader delegates to its parent first; core classes always come from the bootstrap loader, which gives security and uniqueness.

**Q9. How would you debug an OutOfMemoryError in production?**
Enable `HeapDumpOnOutOfMemoryError`, analyze the dump in MAT (dominator tree, path to GC root), check GC logs/metrics, then fix the leak or right-size the heap and container limits.

## Quick revision
- Heap (objects) / Metaspace (classes) / Stack (per-thread frames) / PC / native stack.
- Class loading: bootstrap → platform → application, with parent delegation.
- GC = reachability from roots; young (Eden, S0, S1, minor) → old (major); STW pauses.
- G1 default; ZGC for low latency; Parallel for throughput.
- Leaks = unwanted references (static caches, ThreadLocal, listeners).
- `-Xms/-Xmx`, `MaxRAMPercentage`, heap dumps + MAT, `jstack`, JFR.
