# Introduction to AOP and Cross-Cutting Concerns

> **Source:** Lecture 22 (`Notes_Of_Aditya/Lecture 22/code/aopIntroductionDemo`)
> **Next:** [31_AOP Advice Types](31_AOP%20Advice%20Types.md) · [32_AOP Pointcut Expressions](32_AOP%20Pointcut%20Expressions.md) · [34_Custom Annotations and AOP Proxy Internals](34_Custom%20Annotations%20and%20AOP%20Proxy%20Internals.md)

## Why this topic matters
`@Transactional`, `@Cacheable`, `@Async`, `@PreAuthorize`, and method-level metrics are all **AOP**. Understanding the *why* (scattering and tangling) and the *how* (proxies) explains a lot of Spring's "magic", and its classic bugs such as self-invocation.

---

## 1. Business logic vs supporting logic
```java
public Student createStudent(Student s) { return studentRepository.save(s); }   // business intent: clear
```
Production also needs logging, security checks, timing, auditing, transactions, metrics, retries, and caching. These are **not** about students. They're needed **everywhere**.

- **Concern** = any responsibility (student management, payments, logging, security…).
- **Core concern** = vertical, one per feature (`StudentService`, `OrderService`).
- **Cross-cutting concern** = horizontal, cuts across many features (logging, security, transactions…).

## 2. The two problems
| Problem | Meaning | Symptom |
|---|---|---|
| **Scattering** | One concern spread over many classes | Logging code in 100 methods; a format change means 100 edits; easy to miss one |
| **Tangling** | One method mixes several concerns | Business line buried among log, permission, timer, and audit code |

```java
public Payment processPayment(PaymentRequest r) {
    log.info("Processing payment");                           // logging
    securityService.checkPermission("PROCESS_PAYMENT");       // security
    long start = System.currentTimeMillis();                  // performance
    try {
        Payment p = gateway.charge(r);                        // ← the only business line
        auditService.record("PAYMENT_COMPLETED");             // auditing
        return p;
    } catch (Exception e) { log.error("failed", e); auditService.record("PAYMENT_FAILED"); throw e; }
    finally { metrics.record(System.currentTimeMillis() - start); }
}
```

## 3. Attempts before AOP (and why they fall short)
| Attempt | What it fixes | What remains |
|---|---|---|
| **Utility class** (`LoggingUtil.logStart()`) | Centralizes *what* is done | Calls are still scattered in every method; easy to forget |
| **Inheritance** (`BaseService`) | Shares code | Still called manually; single inheritance; artificial hierarchy |
| **Manual wrapper / Decorator** | Business class stays clean ✅ | One wrapper per interface per concern; manual wiring; nesting explodes |

### The decorator chain from the lecture
```java
public interface StudentService { void createStudent(Student s); }

@Service                                   // real business logic
public class StudentServiceImpl implements StudentService {
    public void createStudent(Student s) { studentRepository.save(s); }
}

@Component                                 // logging wrapper
public class LoggingDecorator implements StudentService {
    private final StudentServiceImpl target;
    public void createStudent(Student s) {
        LoggingServiceUtil.logStart("StudentServiceImpl", "createStudent");
        target.createStudent(s);
        LoggingServiceUtil.logEnd("StudentServiceImpl", "createStudent");
    }
}

@Component @Primary                        // timing wrapper (outermost → injected into controller)
public class ExecutionTimeService implements StudentService {
    private final LoggingDecorator target;
    public void createStudent(Student s) {
        long start = System.currentTimeMillis();
        target.createStudent(s);
        System.out.println(System.currentTimeMillis() - start);
    }
}
```
Chain: `Controller → ExecutionTimeService → LoggingDecorator → StudentServiceImpl → Repository`
Console for one request:
```
Executing -> StudentServiceImpl : createStudent
Student saved
Finishing -> StudentServiceImpl : createStudent
0
```
This is **exactly** the idea of AOP (before → call the real object → after), but done by hand. It doesn't scale: you'd need a wrapper per service type and per concern, wired in the right order, with `@Primary` tricks.

> ⚠️ Lecture code details: the wrappers don't use `try/finally`, so on an exception the "Finishing" log and timing are skipped. Also, the controller's `createStudent(Student student)` has **no `@RequestBody`**, so a JSON body isn't bound (fields come only from query/form parameters).

## 4. What a real solution needs
1. Keep each cross-cutting behaviour in **one module**.
2. **Select where** it applies (all services, methods named `save*`, methods annotated `@TrackExecutionTime`).
3. **Control when** it runs (before, after, on success, on exception, around).
4. **Don't touch** business methods or callers.

## 5. AOP vocabulary
| Term | Meaning | Question |
|---|---|---|
| **Aspect** | Module holding a cross-cutting concern (`@Aspect` class) | — |
| **Advice** | The code that runs (logging, timing…) | **What?** |
| **Pointcut** | Expression selecting join points | **Where?** |
| **Advice type** | `@Before`, `@AfterReturning`, `@AfterThrowing`, `@After`, `@Around` | **When?** |
| **Join point** | A point in execution where advice can apply. In Spring AOP: **a method execution** | — |
| **Target** | The real bean with business logic | — |
| **Proxy** | Spring-generated wrapper that intercepts calls | — |
| **Advisor** | Pointcut + advice pair (internal) | — |
| **Weaving** | Linking aspects to targets. Spring AOP weaves at **runtime** via proxies | — |

## 6. OOP vs AOP: different dimensions
```
OOP (vertical):    StudentService | OrderService | PaymentService       ← business capabilities
AOP (horizontal):  ─────────── Logging / Security / Transactions ────────  ← across all of them
```
They're complementary, not competitors.

## 7. How Spring AOP works: proxies
```
Controller → [Spring AOP proxy] → advice (before) → Target.createStudent() → advice (after) → Controller
```
The container injects the **proxy** where you asked for `StudentService`. You never call `new` on the proxy; Spring builds it during bean creation (BeanPostProcessor). See [34_Custom Annotations and AOP Proxy Internals](34_Custom%20Annotations%20and%20AOP%20Proxy%20Internals.md).

| Manual wrapper | Spring AOP |
|---|---|
| Written per service type | Generic proxy infrastructure |
| Delegation code repeated | Interception logic reusable |
| Assembled by hand (`@Primary`…) | Integrated with the IoC container |
| "Where" hard-coded | Pointcut expressions |
| Many concerns → nested classes | Many aspects on the same method, ordered with `@Order` |

### Spring AOP vs AspectJ
| Spring AOP | AspectJ |
|---|---|
| Runtime proxies (JDK dynamic / CGLIB) | Compile-time / load-time bytecode weaving |
| Method execution join points on **Spring beans** only | Fields, constructors, static/private methods, any object |
| Self-invocation and private methods **not** intercepted | Intercepted |
| Uses AspectJ's annotation syntax (`@Aspect`, pointcut language) | Full language |

---

## Interview Q&A
**Q1. What is AOP and why use it?**
A paradigm for modularizing cross-cutting concerns (logging, security, transactions) separately from business logic, to avoid scattering and tangling.

**Q2. Define aspect, advice, pointcut, and join point.**
Aspect = module of the concern. Advice = code to run. Pointcut = expression choosing where. Join point = an execution point (a method call in Spring AOP).

**Q3. What are scattering and tangling?**
Scattering: one concern spread across many modules. Tangling: one module mixing many concerns.

**Q4. How does Spring implement AOP?**
With runtime proxies (JDK dynamic proxies for interfaces, CGLIB subclasses otherwise) that wrap target beans and run advice around method calls.

**Q5. Spring AOP vs AspectJ?**
Spring AOP is proxy-based, works on Spring beans and method executions only, and is simpler. AspectJ weaves bytecode and can intercept far more (fields, constructors, private methods, self-calls).

**Q6. Which Spring features are built on AOP?**
`@Transactional`, `@Cacheable`, `@Async`, `@Retryable`, method security (`@PreAuthorize`), `@Validated` method validation.

**Q7. How is the decorator pattern related to AOP?**
A decorator manually wraps a target with extra behaviour. AOP generates and applies such wrappers generically, driven by pointcuts.

## Quick revision
- Cross-cutting = horizontal concerns (logging, security, tx…).
- Scattering (one concern, many places) + tangling (one place, many concerns).
- Utility / inheritance / decorator are partial fixes; AOP is the systematic fix.
- Advice = what, pointcut = where, advice type = when, aspect = all of it.
- Spring AOP = runtime proxy around Spring beans.
