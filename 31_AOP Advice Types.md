# AOP Advice Types

> **Source:** Lecture 23 (`Notes_Of_Aditya/Lecture 23/code/aopDemo`)
> **Prerequisite:** [30_Introduction to AOP](30_Introduction%20to%20AOP%20and%20Cross-Cutting%20Concerns.md) · **Next:** [32_AOP Pointcut Expressions](32_AOP%20Pointcut%20Expressions.md)

## Why this topic matters
You must know **when** each advice runs, what it can and cannot change, and why `@Around` is both the most powerful and the most dangerous. Output-ordering questions are common.

---

## 1. Setup
```xml
<!-- Spring Boot 4.x -->
<dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-aspectj</artifactId></dependency>
<!-- Spring Boot 3.x -->
<dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-aop</artifactId></dependency>
```
Boot auto-enables AOP. **No `@EnableAspectJAutoProxy` is needed** (plain Spring needs it).

```java
@Aspect        // this class contains pointcuts + advice
@Component     // AND register it as a bean (@Aspect alone is NOT a component)
public class LoggingAspect { ... }
```
Keep aspects inside the main package so component scanning finds them.

## 2. The try-catch-finally mental model
```java
// @Around wraps all of this
beforeLogic();                         // @Before
try {
    Object result = target();
    afterReturningLogic(result);       // @AfterReturning
    return result;
} catch (Throwable ex) {
    afterThrowingLogic(ex);            // @AfterThrowing
    throw ex;
} finally {
    afterLogic();                      // @After  (= "after finally")
}
```

## 3. `@Before`
```java
@Before("execution(String in.strikes.aopDemo.service.StudentService.createStudent())")
public void logBefore(JoinPoint jp) {
    System.out.println("Entering " + jp.getSignature().getName() + " args=" + Arrays.toString(jp.getArgs()));
}
```
- Runs before the target. **Can stop it only by throwing** an exception (e.g. `SecurityException`).
- **Can't** replace arguments or return an alternative value (use `@Around`).

`JoinPoint` gives you `getSignature()`, `getArgs()`, `getTarget()` (real bean), and `getThis()` (proxy). It can inspect but **not** proceed.

## 4. `@AfterReturning`: success only
```java
@AfterReturning(
    value = "execution(* in.strikes.aopDemo.service.StudentService.createStudent(..))",
    returning = "result")                         // name must match the parameter
public void afterReturning(Student result) {      // parameter type also filters matches
    System.out.println("Returned: " + result.getName());
}
```
- Its return value is **ignored**, so it can't replace the result.
- It *can* mutate a mutable returned object (the lecture sets `result.setName("Rohit")`). That works, but it's **hidden business logic**, which is bad design.
- If the method catches an exception internally and returns normally, `@AfterReturning` runs and `@AfterThrowing` doesn't. Advice reacts to **how the method exits**.

## 5. `@AfterThrowing`: failure only
```java
@AfterThrowing(
    value = "execution(* in.strikes.aopDemo.service.StudentService.createStudent(..))",
    throwing = "exception")
public void afterThrowing(RuntimeException exception) {   // type filters: only RuntimeException (+subclasses)
    System.out.println("Exception type: " + exception.getClass().getName());
    System.out.println("Exception Message: " + exception.getMessage());
}
```
It **observes**; the exception **still propagates** to the caller. It can't return a fallback (its return is ignored). It can throw a different exception.

## 6. `@After`: always (finally)
```java
@After("execution(* in.strikes.aopDemo.service.StudentService.createStudent(..))")
public void after() { System.out.println("logAfterMethod executed"); }
```
| Advice | On success | On exception | Sees result | Sees exception |
|---|---|---|---|---|
| `@AfterReturning` | ✅ | ❌ | ✅ | ❌ |
| `@AfterThrowing` | ❌ | ✅ | ❌ | ✅ |
| `@After` | ✅ | ✅ | ❌ | ❌ |

## 7. `@Around`: full control
```java
@Around("execution(* in.strikes.aopDemo.service.StudentService.createStudent(..))")
public Object around(ProceedingJoinPoint jp) throws Throwable {
    System.out.println("Starting :" + jp.getSignature().getName());
    try {
        Object result = jp.proceed();                 // continue the chain → eventually the target
        System.out.println("Execution Successful");
        return result;                                // ⚠️ MUST return it
    } catch (Exception e) {
        System.out.println("Execution Failed: " + e.getMessage());
        throw e;
    } finally {
        System.out.println("Execution Completed");
    }
}
```
`ProceedingJoinPoint` extends `JoinPoint` and adds `proceed()` / `proceed(Object[] args)`.

What `@Around` can do: run code before and after, **replace arguments**, **replace or wrap the result**, **catch, translate, or swallow exceptions**, **skip the target** (don't call `proceed`), **call it several times**, and measure time.

### Rules
- **Return the result of `proceed()`.** Return `null` and the caller gets `null` even though the target returned a value.
- Declare the return type as `Object` (it handles `String`, `Student`, `int` boxed, and `void` → `null`).
- `proceed()` means "**next in the interceptor chain**", not necessarily the target directly (other aspects or `@Transactional` may be in between).
- Use `finally` for timing so failures are measured too.

### Changing arguments and result (lecture's commented code)
```java
Object[] args = jp.getArgs();
String upper = ((String) args[0]).toUpperCase();
String r = (String) jp.proceed(new Object[]{ upper });   // target receives "ADITYA"
return r + " : String Intercepted";                      // must stay type-compatible with the declared return
```

### ✅ Verified by running the lecture app: calling `proceed()` twice
```java
@Around("execution(* in.strikes.aopDemo.service.StudentService.dummyMethod(..))")
public Object logAroundMethod(ProceedingJoinPoint jp) throws Throwable {
    Object return1 = jp.proceed();
    System.out.println("Intercepted request calliong again");
    Object return2 = jp.proceed();
    return return2;
}
```
`GET /api/students` printed:
```
dummyMethod called
Intercepted request calliong again
dummyMethod called
```
**The target ran twice.** That's harmless here, but for `chargeCard()`, `sendEmail()`, or `createOrder()` it means **duplicate side effects**. Only retry idempotent operations, and prefer Spring Retry or Resilience4j.

### Swallowing exceptions is dangerous
```java
try { return jp.proceed(); }
catch (PaymentGatewayException e) { return new PaymentReceipt("PENDING", "retry later"); }
```
Callers think it succeeded, monitoring misses the failure, and `@Transactional` may commit instead of rolling back.

## 8. Order when several advice types match (one aspect)
```
@Around (before proceed)
  → @Before
    → target
  → @AfterReturning  or  @AfterThrowing
  → @After
@Around (after proceed)
```

## 9. Choose the narrowest advice
| Need | Advice |
|---|---|
| Pre-check / log entry | `@Before` |
| Observe a successful result | `@AfterReturning` |
| Observe a failure | `@AfterThrowing` |
| Cleanup always | `@After` |
| Timing, change args or result, skip, retry, fallback | `@Around` |

---

## Interview Q&A
**Q1. Name the five advice types.**
`@Before`, `@AfterReturning`, `@AfterThrowing`, `@After` (finally), `@Around`.

**Q2. `@After` vs `@AfterReturning`?**
`@After` runs on success **and** failure, like `finally`. `@AfterReturning` runs only on normal return and can read the result.

**Q3. Can `@AfterThrowing` stop the exception from propagating?**
No. It only observes. Use `@Around` with try/catch to translate or swallow it.

**Q4. `JoinPoint` vs `ProceedingJoinPoint`?**
JoinPoint inspects (signature, args, target). ProceedingJoinPoint (only in `@Around`) can also `proceed()`.

**Q5. What happens if `@Around` doesn't call `proceed()`?**
The target (and the rest of the chain) is skipped, and the advice's return value goes to the caller.

**Q6. What if `@Around` forgets to return the result?**
The caller receives `null` (or a `NullPointerException` when unboxing primitives).

**Q7. How do you stop a method from `@Before`?**
Throw an exception.

**Q8. Why use `finally` for timing in `@Around`?**
So the duration is recorded even when `proceed()` throws.

## Quick revision
- Before / AfterReturning / AfterThrowing / After (finally) / Around.
- Only `@Around` can change args or result, skip, or repeat. **Always return `proceed()`'s result.**
- `returning=`/`throwing=` names must match parameters; parameter types filter matches.
- Calling `proceed()` twice runs the target twice (verified).
- Prefer the narrowest advice.
