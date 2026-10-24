# Custom Annotations and AOP Proxy Internals

> **Source:** Lecture 25 (`Notes_Of_Aditya/Lecture 25/code/aopDemo`)
> **Related:** [13_Bean Lifecycle](13_Bean%20Lifecycle.md) · [31_AOP Advice Types](31_AOP%20Advice%20Types.md) · [32_AOP Pointcut Expressions](32_AOP%20Pointcut%20Expressions.md)

## Why this topic matters
"Build a `@TrackExecutionTime` / `@Audit` / `@RateLimit` annotation with AOP" is a very common practical interview task. You must also explain **how Spring creates the proxy** and **why self-invocation breaks it** (the same reason `@Transactional` sometimes "doesn't work").

---

## 1. Declarative: separate "what" from "how"
```
Annotation      → declares WHAT behaviour a method needs   (@TrackExecutionTime)
Aspect          → defines HOW that behaviour works          (timing + logging)
Business method → contains only business logic
```
> An annotation by itself **does nothing**. It's just metadata. Something (an aspect, a BeanPostProcessor, a framework) must detect it and act.

## 2. Creating the annotation (lecture code)
```java
@Target(ElementType.METHOD)            // WHERE it may be placed
@Retention(RetentionPolicy.RUNTIME)    // HOW LONG it survives (must be RUNTIME for Spring to see it)
@Documented                            // include in Javadoc (no effect on AOP)
public @interface TrackExecutionTime {
    long warnAfter() default 2000;     // configuration properties with defaults
    String operation() default "";
}
```
| `@Retention` | Kept in | Visible to Spring at runtime? |
|---|---|---|
| `SOURCE` | Source only (e.g. `@Override`) | ❌ |
| `CLASS` (**default**) | `.class` file | ❌ |
| `RUNTIME` | Loaded class | ✅ required |

`@Target` options: `METHOD`, `TYPE`, `FIELD`, `PARAMETER`, `CONSTRUCTOR`, `ANNOTATION_TYPE`, and combinations like `{TYPE, METHOD}`.
**Marker annotation** = no properties. **Configured annotation** = has properties (like this one).

## 3. Using it
```java
@Service
public class StudentService {

    @TrackExecutionTime(warnAfter = 2000, operation = "Creating New Student")
    public Student createStudent(Student student) {
        System.out.println("Student saved");
        return student;
    }

    @TrackExecutionTime(warnAfter = 1500, operation = "Get Student Data")
    public String getStudent(String s) {
        try { Thread.sleep(2000); } catch (Exception e) {}   // simulate slowness
        System.out.println(s);
        return s;
    }
}
```

## 4. The aspect: binding the annotation
```java
@Component
@Aspect
public class SimpleAspect {

    @Around("@annotation(trackExecutionTime)")                 // BINDING form: lowercase = parameter name
    public Object measureExecutionTime(ProceedingJoinPoint joinPoint,
                                       TrackExecutionTime trackExecutionTime) throws Throwable {
        long startTime = System.currentTimeMillis();
        try {
            return joinPoint.proceed();
        } finally {                                            // measured on success AND failure
            long duration = System.currentTimeMillis() - startTime;
            String operation = trackExecutionTime.operation();
            if (operation.isBlank()) operation = joinPoint.getSignature().getName();
            if (duration >= trackExecutionTime.warnAfter())
                System.out.println("SLOW OPERATION ALERT : Time Taken by " + operation + ": " + duration);
            else
                System.out.println("Time Taken by " + operation + ": " + duration);
        }
    }
}
```
| Form | Example | Gives you |
|---|---|---|
| Type matching | `@annotation(com.x.TrackExecutionTime)` | Only "is it present?" |
| **Binding** | `@annotation(trackExecutionTime)` + parameter `TrackExecutionTime trackExecutionTime` | The annotation **instance**, so you can read `operation()` and `warnAfter()` |

### ✅ Verified by running the lecture app
```
POST /api/students  → Student saved
                      Time Taken by Creating New Student: 0
GET  /api/students  → All Student Data
                      SLOW OPERATION ALERT : Time Taken by Get Student Data: 2001
```
`getStudent` sleeps 2000 ms, which is more than `warnAfter = 1500`, so it's flagged as slow.
Improvement: use `System.nanoTime()` for elapsed time and an SLF4J logger instead of `System.out`.

## 5. Where AOP enters the bean lifecycle
```
1. Read BeanDefinition
2. Instantiate target
3. Inject dependencies
4. Aware callbacks
5. BeanPostProcessor.postProcessBeforeInitialization
6. Init callbacks (@PostConstruct, afterPropertiesSet, initMethod)
7. BeanPostProcessor.postProcessAfterInitialization   ← AOP auto-proxy creator works HERE
      "Does any advisor (pointcut + advice) match this bean?"
         no  → return the original bean
         yes → return a PROXY wrapping the bean
8. Container stores and injects the FINAL reference (the proxy)
```
The auto-proxy creator (`AnnotationAwareAspectJAutoProxyCreator`) is itself a **BeanPostProcessor**. Advice doesn't run at creation time; it runs later when methods are **called through the proxy**.

```
StudentController.studentService ──► [Proxy P200] ──► [Target T100: StudentService]
```
The bean name `studentService` resolves to the proxy, and the target sits behind it.

### BeanPostProcessor
```java
public interface BeanPostProcessor {
    default Object postProcessBeforeInitialization(Object bean, String name) { return bean; }
    default Object postProcessAfterInitialization(Object bean, String name)  { return bean; }   // may return a proxy
}
```

## 6. Interceptor chain and `@Order`
Several aspects on one method form a nested chain:
```
Caller → Proxy → Security → Logging → Performance → Target
                                                     ↓
Caller ← Proxy ← Security ← Logging ← Performance ←──┘
```
```java
@Aspect @Component @Order(1) class SecurityAspect {}     // lowest value = OUTERMOST = runs first on the way in
@Aspect @Component @Order(2) class LoggingAspect {}
@Aspect @Component @Order(3) class PerformanceAspect {}
```
`proceed()` = "go to the **next** interceptor"; only the last `proceed()` reaches the target. Without `@Order`, the order between aspects is undefined.

## 7. ⚠️ Proxy-based AOP limitations (very common interview topic)
1. **Self-invocation bypasses the proxy:**
   ```java
   @Service
   public class ReportService {
       public void generateAll() { generateAnnual(); }        // this.generateAnnual() → NOT intercepted
       @TrackExecutionTime public void generateAnnual() {}
   }
   ```
   The call never leaves the target object. Fixes: move the method to another bean; inject the bean into itself (`@Lazy` self-reference) or use `ObjectProvider`; `AopContext.currentProxy()` with `exposeProxy=true`; or AspectJ weaving.
   **Same reason `@Transactional` on an internally-called method is ignored.**
2. **Private methods** aren't intercepted (proxies can't override them). Final methods and classes can't be intercepted with CGLIB.
3. **Objects created with `new`** aren't Spring beans, so they have no proxy and no advice.
4. Java doesn't inherit **method annotations declared only on an interface**, so `@annotation(...)` may not match the implementation's method. Put custom AOP annotations on the implementation class's method.

---

## Interview Q&A
**Q1. How would you build a custom `@LogExecutionTime` annotation?**
Declare it with `@Target(METHOD)` and `@Retention(RUNTIME)`, write an `@Aspect @Component` with `@Around("@annotation(logExecutionTime)")`, time `proceed()` inside `try/finally`, and return the result.

**Q2. Why must retention be RUNTIME?**
Spring reads annotations via reflection at runtime. With `CLASS` or `SOURCE`, they're invisible.

**Q3. How do you read annotation values inside advice?**
Bind it: `@annotation(myAnno)` plus a method parameter `MyAnno myAnno`.

**Q4. When does Spring create AOP proxies?**
During bean creation, in `postProcessAfterInitialization` of the auto-proxy-creator BeanPostProcessor, and only for beans matched by some advisor.

**Q5. Why doesn't my aspect or `@Transactional` work when calling a method from the same class?**
Self-invocation uses `this`, not the proxy, so interception never happens.

**Q6. How do you order multiple aspects?**
`@Order(n)` or `Ordered`. Lower values are outer and run first before the target, last after it.

**Q7. Can Spring AOP intercept private methods?**
No (and not final methods with CGLIB). Use AspectJ weaving if needed.

**Q8. What is a BeanPostProcessor?**
A container hook that can inspect, modify, or replace beans before and after initialization. AOP proxying, `@Autowired` processing, and `@PostConstruct` handling all use it.

## Quick revision
- Annotation = metadata (what); aspect = behaviour (how).
- `@Target(METHOD)` + **`@Retention(RUNTIME)`** are essential.
- `@annotation(param)` binds the instance so you can read its properties.
- The proxy is created in `postProcessAfterInitialization`; the container injects the proxy.
- `@Order`: lower = outer. Self-invocation, private, final, and `new` objects aren't advised.
