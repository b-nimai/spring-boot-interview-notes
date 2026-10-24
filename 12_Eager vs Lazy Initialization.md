# Eager vs Lazy Initialization

> **Source:** Lecture 06 (`Notes_Of_Aditya/Lecture 06/code/BeanInitializationDemo`), Lecture 07 (`@Lazy` and lifecycle)
> **Practice:** `Notes_Of_Aditya/Lecture 06/PracticeQuestions.pdf`

## Why this topic matters
This is about **when** Spring creates a bean. Output-prediction questions ("what prints first?") are built on this.

---

## 1. Defaults
| Bean type | Created |
|---|---|
| Singleton | **Eagerly**: when the context starts |
| Prototype | **Lazily**: each time it is requested |
| `@Lazy` singleton | When first requested (or needed by someone) |

## 2. Eager initialization (default for singletons)
```java
@Component
public class PaymentService {
    public PaymentService() { System.out.println("PaymentService object created"); }
}
// new AnnotationConfigApplicationContext(AppConfig.class) → prints immediately
```
**Why eager is the default: fail fast.** Missing beans, bad config, or broken wiring are discovered **at startup**, not when the first user hits a feature. It also gives predictable startup validation.

## 3. Lazy initialization
```java
@Component
@Lazy
public class ReportService {
    public ReportService() { System.out.println("ReportService object created"); }
    public void generateReport() { System.out.println("Report generated"); }
}
```
```java
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
System.out.println("Application started");
ReportService rs = context.getBean(ReportService.class);
rs.generateReport();
```
Output:
```
Application started
ReportService object created
Report generated
```
At startup Spring keeps only the **BeanDefinition**. The object, its DI, and its init callbacks all happen later.

## 4. ⚠️ Lazy bean needed by an eager bean
```java
@Component                    // eager singleton
public class UserService {
    public UserService(ReportService reportService) { ... }   // ReportService is @Lazy
}
```
The eager bean needs it now, so **ReportService is still created at startup**. `@Lazy` on the class is ignored in practice.

## 5. `@Lazy` at the injection point: proxy
```java
@Component
public class UserService {
    private final EmailService emailService;
    public UserService(@Lazy EmailService emailService) {      // a PROXY is injected
        this.emailService = emailService;
        System.out.println("UserService created");
    }
    public void registerUser() {
        System.out.println("User registered");
        emailService.sendEmail();                               // real bean resolved HERE
    }
}
```
Output:
```
UserService created
Application started
User registered
EmailService created
Email sent
```
| Usage | Meaning |
|---|---|
| `@Lazy` on **class** | Don't create this bean until it's requested |
| `@Lazy` on **injection point** | Inject a **proxy** and resolve the real bean on first method call |

This is also a workaround for circular dependencies (see [Circular Dependency](10_Circular%20Dependency.md)).

## 6. Scenarios
| Scenario | Result |
|---|---|
| Eager → Lazy (direct) | Lazy bean created at startup anyway |
| Eager → `@Lazy` injection point | Proxy injected; real bean created on first use |
| Lazy → Eager | Eager created at startup; lazy created later and receives the existing eager bean |
| Lazy → Lazy | Neither at startup; first request creates both (unless the injection point is `@Lazy`) |

## 7. Global lazy init in Spring Boot
```properties
spring.main.lazy-initialization=true     # default false
```
Opt a bean back out:
```java
@Component
@Lazy(false)
public class ImportantStartupBean { }
```
**Trade-off:** faster startup and less memory up front, but config errors appear **at runtime** and the first request is slower.

---

## Interview Q&A
**Q1. When are singleton beans created?**
Eagerly, during context startup, unless marked `@Lazy`.

**Q2. Why is eager the default?**
Fail fast: configuration and wiring errors show up at startup.

**Q3. If a `@Lazy` bean is a dependency of an eager bean, when is it created?**
At startup, because the eager bean needs it, unless the injection point itself is `@Lazy` (a proxy is injected).

**Q4. Class-level `@Lazy` vs injection-point `@Lazy`?**
Class-level delays creating the bean. Injection-point injects a lazy-resolution proxy.

**Q5. How do you make all beans lazy in Spring Boot? Downside?**
`spring.main.lazy-initialization=true`. Errors surface late and the first requests are slower.

**Q6. Are prototype beans eager or lazy?**
Always created on demand, so effectively lazy.

## Quick revision
- Singleton = eager (fail fast). Prototype = on demand.
- `@Lazy` class = delay creation; `@Lazy` param = proxy.
- An eager bean that depends on a lazy one forces it to be created at startup.
- `spring.main.lazy-initialization=true` makes all beans lazy; opt out with `@Lazy(false)`.
