# Exceptions

> **Part of:** Core Java (51–62) · **Related:** [25_Exception Handling in Spring](25_Exception%20Handling%20and%20Global%20Exception%20Handler.md), [42_Transactions rollback rules](42_Transactions%20and%20ACID.md)

## Why this topic matters
Checked vs unchecked, `finally` semantics, try-with-resources, and custom exceptions come up constantly, and they directly affect Spring transaction rollback.

---

## 1. Hierarchy
```
Throwable
 ├─ Error                      (JVM/serious: OutOfMemoryError, StackOverflowError) — don't catch
 └─ Exception
     ├─ checked                (IOException, SQLException, InterruptedException, ClassNotFoundException)
     └─ RuntimeException       (unchecked: NullPointerException, IllegalArgumentException,
                                IllegalStateException, IndexOutOfBounds, ArithmeticException,
                                ClassCastException, NumberFormatException, ConcurrentModificationException,
                                UnsupportedOperationException)
```
| | Checked | Unchecked |
|---|---|---|
| Compiler | **Must** catch or declare `throws` | Not enforced |
| Meaning | Recoverable external conditions (I/O, network) | Programming bugs / invalid state |
| Spring `@Transactional` | **Commits** by default | **Rolls back** by default |
| Modern trend | Wrapped into unchecked (Spring's `DataAccessException`) | Preferred for business exceptions |
> 🟦 TS has no checked exceptions; anything can be thrown. Java forces you to deal with checked ones at compile time.

## 2. try / catch / finally
```java
try {
    riskyCall();
} catch (FileNotFoundException e) {        // specific first
    log.warn("missing", e);
} catch (IOException | ParseException e) { // multi-catch (Java 7); e is effectively final
    throw new ConfigException("Bad config", e);   // wrap and KEEP THE CAUSE
} finally {
    cleanup();                             // always runs (except System.exit / JVM crash / infinite loop)
}
```
- Catch order: subclass before superclass (otherwise a compile error: unreachable).
- **`finally` overriding a return**: if both `try` and `finally` return, the `finally` value wins; a `return` in `finally` also **swallows exceptions**. Never return from `finally`.
```java
int f() { try { return 1; } finally { return 2; } }   // returns 2 (bad practice)
int g() { int x = 1; try { return x; } finally { x = 5; } }   // returns 1 (value already evaluated)
```

## 3. try-with-resources (Java 7)
```java
try (Connection c = ds.getConnection();
     PreparedStatement ps = c.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {
    ...
}   // closed automatically in REVERSE order, even on exceptions
```
- Works with any `AutoCloseable`.
- If the body throws and `close()` also throws, the body's exception is primary and the close exception is attached as **suppressed** (`e.getSuppressed()`). The old finally-based approach lost the original.
- Java 9: effectively-final variables can be used directly: `try (conn) { … }`.

## 4. throw vs throws; custom exceptions
```java
public class InsufficientBalanceException extends RuntimeException {   // business rule → unchecked
    private final BigDecimal shortBy;
    public InsufficientBalanceException(BigDecimal shortBy) {
        super("Insufficient balance, short by " + shortBy);
        this.shortBy = shortBy;
    }
    public BigDecimal getShortBy() { return shortBy; }
}

public void withdraw(BigDecimal amt) {                      // no 'throws' needed (unchecked)
    if (amt.compareTo(balance) > 0) throw new InsufficientBalanceException(amt.subtract(balance));
}
public String read(Path p) throws IOException { return Files.readString(p); }   // checked → declared
```
- `throw` = raise an instance. `throws` = declare in the signature.
- **Exception chaining**: `new XException("context", cause)` keeps the root cause in the stack trace.
- Overriding rule: an overriding method can't throw **broader checked** exceptions than the parent.

## 5. Best practices
- **Don't swallow**: `catch (Exception e) {}` hides bugs. At least log, rethrow, or handle meaningfully.
- Don't catch `Throwable`/`Error`. Avoid catching generic `Exception` except at boundaries (`@RestControllerAdvice`, the top-level runner).
- Log **or** rethrow, not both at every layer (duplicate stack traces).
- Use exceptions for exceptional cases, not normal control flow (they're expensive: stack trace capture).
- Prefer standard exceptions: `IllegalArgumentException` (bad argument), `IllegalStateException` (wrong state), `UnsupportedOperationException`, `NoSuchElementException`.
- Validate early: `Objects.requireNonNull(x, "x must not be null")`.
- Restore the interrupt flag: `catch (InterruptedException e) { Thread.currentThread().interrupt(); }`.
- In Spring: business exceptions extend `RuntimeException` and are mapped to HTTP codes centrally; checked exceptions need `rollbackFor`.

## 6. NullPointerException and `Optional`
Java 14+ **helpful NPE messages**: `Cannot invoke "String.length()" because "user.name" is null`.
Avoid NPEs with `Optional` return types ([58](58_Java%208%2B%20-%20Lambdas%2C%20Functional%20Interfaces%2C%20Streams%20and%20Optional.md)), `Objects.requireNonNullElse`, and null-safe comparisons (`"x".equals(s)`).

## 7. Errors you should recognize
| Error | Cause |
|---|---|
| `StackOverflowError` | Unbounded recursion / circular calls |
| `OutOfMemoryError: Java heap space` | Leak or too-small heap |
| `OutOfMemoryError: Metaspace` | Too many classes (classloader leaks) |
| `NoClassDefFoundError` | Class present at compile time, missing/failed init at runtime |
| `ClassNotFoundException` (checked) | Dynamic loading (`Class.forName`) can't find the class |
| `ExceptionInInitializerError` | Exception in a static initializer |

---

## Interview Q&A
**Q1. Checked vs unchecked exceptions?**
Checked must be handled or declared (recoverable external conditions). Unchecked are `RuntimeException`s (programming errors), not enforced by the compiler.

**Q2. Does `finally` always execute?**
Yes, except on `System.exit`, a JVM crash/kill, or an infinite loop/deadlock in try/catch.

**Q3. What if both `try` and `finally` return a value?**
`finally`'s return wins and any exception is swallowed. Avoid it.

**Q4. What is try-with-resources? What are suppressed exceptions?**
Automatic closing of `AutoCloseable`s in reverse order. Exceptions from `close()` are attached as suppressed to the primary exception.

**Q5. `throw` vs `throws`?**
Raise an exception vs declare a checked exception in the method signature.

**Q6. `ClassNotFoundException` vs `NoClassDefFoundError`?**
A checked exception from dynamic loading vs an error when a class known at compile time is missing or failed to initialize at runtime.

**Q7. Should custom exceptions be checked or unchecked?**
Usually unchecked for business/domain errors (cleaner APIs, Spring rollback). Checked when callers can and must recover.

**Q8. Why does Spring convert `SQLException` into `DataAccessException`?**
Unchecked + vendor-neutral: no `throws` everywhere, and a consistent exception hierarchy.

## Quick revision
- Throwable → Error | Exception → (checked) | RuntimeException (unchecked).
- Specific catch first; multi-catch; always chain the cause.
- `finally` always runs; never `return` from it.
- try-with-resources closes in reverse order; suppressed exceptions are preserved.
- Business exceptions are unchecked; Spring rolls back on unchecked by default.
