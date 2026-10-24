# Modern Java 9–21 Features

> **Part of:** Core Java (51–62) · Spring Boot 3 requires **Java 17+**; Boot 4 targets 17+ and works best on **21 (LTS)**.

## Why this topic matters
Interviewers check whether you write modern Java: records, sealed classes, pattern matching, switch expressions, text blocks, `var`, and especially **virtual threads** (Java 21). LTS versions: **8, 11, 17, 21, 25**.

---

## 1. Version cheat sheet
| Version | Key features |
|---|---|
| **9** | Modules (JPMS), `List.of/Set.of/Map.of`, private interface methods, `Optional.or/stream/ifPresentOrElse`, Stream `takeWhile/dropWhile/iterate`, JShell |
| **10** | `var` (local type inference), `List.copyOf`, `Optional.orElseThrow()` |
| **11 (LTS)** | `String.isBlank/strip/lines/repeat`, `Files.readString/writeString`, standard `HttpClient`, `var` in lambdas, single-file `java Hello.java` |
| **14** | **Switch expressions** (final), helpful NullPointerException messages |
| **15** | **Text blocks** |
| **16** | **Records**, pattern matching for `instanceof`, `Stream.toList()` |
| **17 (LTS)** | **Sealed classes**, strong encapsulation of JDK internals |
| **21 (LTS)** | **Virtual threads**, **pattern matching for switch** + **record patterns**, **sequenced collections**, generational ZGC |
| **25 (LTS)** | Scoped values, structured concurrency progress, compact source files / instance main, flexible constructor bodies |

## 2. `var` (10)
```java
var users = new ArrayList<User>();        // inferred ArrayList<User>
for (var e : map.entrySet()) { ... }
```
Local variables only (not fields, params, or returns), and it must have an initializer. Still statically typed. Use it when the type is obvious from the right side.

## 3. Switch expressions (14)
```java
String type = switch (day) {
    case SATURDAY, SUNDAY -> "Weekend";            // arrow: no fall-through, no break
    case MONDAY -> { log("start"); yield "Workday"; }   // block → yield
    default -> "Workday";
};
```
They return a value, have no fall-through, and must be **exhaustive** (enums/sealed types can skip `default` when all cases are covered).

## 4. Text blocks (15)
```java
String json = """
        {
          "name": "%s",
          "age": %d
        }
        """.formatted(name, age);
String sql = """
        SELECT id, name FROM students
        WHERE email = :email
        """;
```
Incidental indentation is stripped; `\` at line end = no newline; `\s` = preserved space.

## 5. Records (16)
Concise immutable data carriers. See [53](53_equals%2C%20hashCode%2C%20Immutability%20and%20Records.md).
```java
public record Money(BigDecimal amount, String currency) {
    public Money { if (amount.signum() < 0) throw new IllegalArgumentException(); }
}
```

## 6. Pattern matching for `instanceof` (16)
```java
// before
if (obj instanceof String) { String s = (String) obj; use(s); }
// now
if (obj instanceof String s && !s.isBlank()) use(s);
```

## 7. Sealed classes (17)
Restrict which classes can extend or implement a type, giving a closed hierarchy (algebraic data types).
```java
public sealed interface Payment permits CardPayment, UpiPayment, WalletPayment {}
public record CardPayment(String cardNo) implements Payment {}
public record UpiPayment(String vpa) implements Payment {}
public non-sealed class WalletPayment implements Payment {}   // subclasses must be final, sealed, or non-sealed
```
Benefits: the domain model is explicit, and the compiler knows **all** subtypes, so exhaustive switches don't need `default` (adding a subtype makes the compiler flag every switch).

## 8. Pattern matching for switch + record patterns (21)
```java
BigDecimal fee(Payment p) {
    return switch (p) {
        case CardPayment c when c.cardNo().startsWith("4") -> new BigDecimal("2.0");   // guard
        case CardPayment c   -> new BigDecimal("2.5");
        case UpiPayment u    -> BigDecimal.ZERO;
        case WalletPayment w -> new BigDecimal("1.0");
        // no default needed: sealed + exhaustive
    };
}
// record deconstruction
if (shape instanceof Rectangle(Point(var x1, var y1), Point(var x2, var y2))) { area = (x2 - x1) * (y2 - y1); }
Object o = ...;
String desc = switch (o) {
    case null -> "null";                  // switch can now handle null
    case Integer i when i > 10 -> "big int";
    case Integer i -> "int";
    case String s -> "string of length " + s.length();
    default -> "other";
};
```
This is a modern alternative to the Visitor pattern in LLD.

## 9. Virtual threads (21): Project Loom
**Problem:** platform threads map 1:1 to OS threads (~1 MB stack each, expensive), so thread-per-request servers top out at a few thousand concurrent requests. Reactive programming (WebFlux) solves this but makes code hard to read and debug.
**Virtual threads** are lightweight threads managed by the JVM, **mounted on a few carrier (platform) threads**. When a virtual thread blocks on I/O, the JVM **unmounts** it and the carrier runs another one.
```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 100_000).forEach(i ->
        executor.submit(() -> { Thread.sleep(Duration.ofSeconds(1)); return i; }));   // 100k concurrent sleeps, cheap
}
Thread.startVirtualThread(() -> handle());
Thread.ofVirtual().name("worker-", 0).start(task);
```
> ✅ Verified on JDK 25: **10,000 virtual threads each sleeping 1 s finished in ~1.03 s total.** A fixed pool of 200 platform threads would need ~50 s.

```properties
# Spring Boot 3.2+: Tomcat request handling, @Async, schedulers on virtual threads
spring.threads.virtual.enabled=true
```
| | Platform threads | Virtual threads |
|---|---|---|
| Cost | ~1 MB stack, OS-managed | KBs, heap-allocated, JVM-managed |
| Count | Thousands | Millions |
| Best for | CPU-bound work | **I/O-bound**, blocking-style code |
| Pool them? | Yes | **No**: create one per task |
Caveats:
- No speedup for CPU-bound work.
- **Pinning**: blocking inside `synchronized` pins the carrier on Java 21–23 (fixed in **24**). Prefer `ReentrantLock` on 21.
- Avoid huge `ThreadLocal`s (millions of threads).
- The **downstream** limits still apply (DB connection pool size!).
> 🟦 Node gets concurrency from a single-threaded event loop + async I/O. Virtual threads give Java the same scalability while keeping simple **blocking-style** code (no `async/await` colouring).

Related previews: **structured concurrency** (`StructuredTaskScope`, treat related tasks as one unit) and **scoped values** (an immutable, cheaper ThreadLocal alternative).

## 10. Sequenced collections (21)
```java
List<Integer> list = ...;  list.getFirst(); list.getLast(); list.reversed();
LinkedHashMap<K,V> m = ...; m.firstEntry(); m.pollLastEntry(); m.putFirst(k, v);
```
A uniform API for collections with a defined encounter order (`SequencedCollection`, `SequencedSet`, `SequencedMap`).

## 11. Other handy APIs
```java
HttpClient client = HttpClient.newHttpClient();                       // Java 11 HTTP client (HTTP/2, async)
HttpResponse<String> res = client.send(HttpRequest.newBuilder(URI.create(url)).build(),
                                       HttpResponse.BodyHandlers.ofString());
String content = Files.readString(Path.of("a.txt"));
"  hi ".strip(); "".isBlank(); "ab".repeat(3); text.lines().count();
Stream.iterate(1, i -> i <= 100, i -> i * 2).toList();
Collectors.teeing(min, max, (a, b) -> ...);                            // 12
String s = "Hello %s".formatted(name);                                // 15
Math.clamp(v, 0, 10);                                                  // 21
```
**JPMS modules** (`module-info.java`, `requires`/`exports`): strong encapsulation. Rarely used in Spring Boot apps, but know that it exists.

---

## Interview Q&A
**Q1. What's new in Java 17 vs 11 vs 21 that you've used?**
11: String/Files/HttpClient APIs. 17: records, sealed classes, pattern-matching instanceof, switch expressions, text blocks. 21: virtual threads, pattern-matching switch, record patterns, sequenced collections.

**Q2. What are virtual threads, and when do they help?**
JVM-managed lightweight threads multiplexed onto carrier threads. They let blocking I/O code scale to huge concurrency; they don't help CPU-bound work.

**Q3. Virtual threads vs reactive (WebFlux)?**
Both handle high I/O concurrency. Virtual threads keep imperative, blocking-style code and normal stack traces/debugging; reactive needs non-blocking libraries end to end.

**Q4. What is thread pinning?**
A virtual thread that can't unmount (blocking inside `synchronized` before Java 24, or native calls) and so holds its carrier thread.

**Q5. What do sealed classes give you?**
Controlled inheritance plus compiler-checked exhaustive pattern matching.

**Q6. Switch expression vs switch statement?**
An expression returns a value, arrow labels don't fall through, and it must be exhaustive. `yield` returns from blocks.

**Q7. `record` vs Lombok `@Value`?**
Both give immutable data classes. Records are a language feature (pattern matching, deconstruction) with no dependency; Lombok offers builders and more customization.

**Q8. Why does Spring Boot 3 require Java 17?**
Jakarta EE 9+ namespace, modern language features, and the baseline for AOT/native support.

## Quick revision
- LTS: 8, 11, 17, 21, 25.
- `var`, switch expressions (`->`, `yield`, exhaustive), text blocks, records, `instanceof` patterns.
- Sealed + pattern-matching switch + record patterns = exhaustive, type-safe domain logic.
- Virtual threads (21): cheap, for blocking I/O, don't pool them, watch pinning (synchronized before 24) and DB pool limits; `spring.threads.virtual.enabled=true`.
- Sequenced collections: `getFirst`/`getLast`/`reversed`.
