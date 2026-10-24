# AOP Pointcut Expressions

> **Source:** Lecture 24 (`Notes_Of_Aditya/Lecture 24/code/aopDemo`)
> **Practice:** [33_AOP Pointcut Practice Questions](33_AOP%20Pointcut%20Practice%20Questions.md)

## Why this topic matters
A pointcut decides **where** advice applies. Too broad means performance problems and surprises; too narrow means silently missed methods. Interviewers ask you to read and write `execution(...)` expressions and explain `this` vs `target`.

---

## 1. `execution()`: match by method signature
```
execution( modifiers? return-type declaring-type? method-name(params) throws? )
```
Required: **return type, method name, params**. `?` parts are optional.

```java
execution(public Student com.coderarmy.student.service.StudentService.getStudent(java.lang.Long))  // exact
execution(* *(..))                       // any method anywhere (too broad!)
```

### Wildcards
| Pattern | Meaning |
|---|---|
| `*` (return type) | any return type |
| `get*` / `*Student` / `*Student*` | name starts with / ends with / contains (case-sensitive) |
| `()` | **zero** parameters |
| `(*)` | **exactly one** parameter of any type |
| `(*,*)` | exactly two |
| `(..)` | **zero or more** parameters |
| `(java.lang.Long, ..)` | first param Long, then anything |
| `(.., com.x.Student)` | last param Student |
| `service.*` | types **directly** in package `service` |
| `service..*` | types in `service` **and all sub-packages** |

```java
execution(* com.coderarmy.student.service..*.*(..))          // every method in the service layer
execution(public * com.coderarmy.student.service..*.get*(..)) // public getters in the service layer
execution(void *(..))                                        // methods returning void
execution(com.coderarmy.student.model.Student *(..))         // returning exactly Student (not List<Student>)
```

## 2. Other designators
| Designator | Checks | Example |
|---|---|---|
| `within(type)` | Declaring class/package | `within(com.coderarmy.student.service..*)` |
| `@annotation(A)` | Annotation **on the method** | `@annotation(com.x.TrackExecutionTime)` |
| `@within(A)` | Annotation on the **declaring type** | `@within(org.springframework.stereotype.Service)` |
| `@target(A)` | Annotation on the **runtime target's class** | `@target(com.x.Audited)` |
| `bean(name)` | **Spring bean name** (Spring-only) | `bean(studentService)`, `bean(*Service)` |
| `args(T)` | Runtime argument types | `args(com.x.Student)`, `args(Long, ..)` |
| `@args(A)` | Annotation on the **runtime argument's class** | `@args(com.x.SensitiveData)` |
| `this(T)` | **Proxy** type | `this(com.x.StudentOperations)` |
| `target(T)` | **Real bean** type | `target(com.x.StudentService)` |

`execution` vs `within`: `within` only restricts *where the method is declared*. `execution` can also filter by modifier, return type, name, and parameters. They combine:
```java
within(com.coderarmy.student.service..*) && execution(public * get*(..))
```

### `@within` vs `@target` (inheritance)
```java
public class BaseService { public void save() {} }       // not annotated
@TrackExecution @Service
public class StudentService extends BaseService {}
// studentService.save():
//   @target(TrackExecution) ✅ runtime object is StudentService (annotated)
//   @within(TrackExecution) ❌ save() is DECLARED in un-annotated BaseService
```

### `args` vs `@args`
`args(Student)` matches the runtime argument type. `@args(TrackData)` matches when the argument's **class** is annotated `@TrackData` (not a parameter annotation).

## 3. Combining: `&&`, `||`, `!`
```java
within(com.x.service..*) && execution(public * *(..))           // public service methods
within(com.x.controller..*) || within(com.x.service..*)         // either layer
within(com.x..*) && !within(com.x.repository..*)                // everything except repositories
within(com.x.service..*) && !execution(* delete*(..))           // service minus deletes
```

## 4. Named pointcuts: lecture's `ApplicationPointcuts`
```java
public class ApplicationPointcuts {

    @Pointcut("within(in.strikes.aopDemo.controller..*)")
    public void controllerLayer() {}            // empty body: the NAME represents the expression

    @Pointcut("within(in.strikes.aopDemo.service..*)")
    public void serviceLayer() {}

    @Pointcut("execution(public * * (..))")
    public void publicMethod() {}

    @Pointcut("serviceLayer() && publicMethod()")   // compose named pointcuts
    public void publicServiceMethod() {}

    @Pointcut("execution(* *.get* (..))")
    public void getterMethod() {}
}

// use from an aspect with the fully-qualified name:
@Before("in.strikes.aopDemo.aspect.ApplicationPointcuts.publicServiceMethod()")
public void log() { ... }
```
One place defines the architecture. If a package moves, you edit one line.

## 5. JDK dynamic proxy vs CGLIB proxy
| | JDK dynamic proxy | CGLIB proxy |
|---|---|---|
| How | Implements the bean's **interfaces** | **Subclasses** the bean's class |
| Needs | An interface | Non-final class; non-final, non-private methods |
| Injectable as concrete class? | ❌ only as the interface | ✅ |
| `this(StudentService)` (class) | ❌ proxy isn't a `StudentService` | ✅ |
| `this(StudentOperations)` (interface) | ✅ | ✅ |
| `target(StudentService)` | ✅ | ✅ |

Spring Boot default: **CGLIB** (`spring.aop.proxy-target-class=true`). Plain Spring `@EnableAspectJAutoProxy(proxyTargetClass = false)` defaults to JDK proxies when interfaces exist.

### ✅ Verified by running the lecture app
Lecture 24 sets:
```properties
spring.aop.proxy-target-class=false     # prefer JDK interface proxies
```
```java
@Service
public class StudentService implements StudentServiceInterface { ... }

@Before("this(in.strikes.aopDemo.service.StudentService)")    // concrete CLASS
public void logBeforeMethod4() { System.out.println("Method Intercepted"); }
```
Calling `POST` and `GET /api/students` printed `Student saved` and `All Student Data`, but **never "Method Intercepted"**. The JDK proxy implements only `StudentServiceInterface`; it isn't a `StudentService`, so `this(StudentService)` can't match.
Fixes: use `target(...StudentService)`, use `this(...StudentServiceInterface)`, or switch back to CGLIB (`proxy-target-class=true`).
(The controller also injects `StudentServiceInterface`, which is required: with a JDK proxy, injecting the concrete class fails.)

## 6. Practical guidelines
1. Prefer architectural boundaries (`within(..service..*)`) over `execution(* *(..))`.
2. Use `execution` when the name, return type, or params matter.
3. Use `@annotation` for **opt-in** behaviour (explicit and safe).
4. Name and centralize pointcuts.
5. Remember the proxy limits: no **self-invocation**, no **private** methods, beans only.
6. Pointcut matching is **case-sensitive**: `*Student*` does **not** match `studentExists`.

---

## Interview Q&A
**Q1. Explain `execution(* com.x.service..*.*(..))`.**
Any return type, any class in `com.x.service` or its sub-packages, any method name, any parameters.

**Q2. `(*)` vs `(..)`?**
Exactly one parameter vs zero or more.

**Q3. `within` vs `execution`?**
`within` limits by declaring type/package. `execution` matches a full method signature pattern.

**Q4. `this` vs `target`?**
`this` = the proxy object's type. `target` = the underlying bean's type. They differ with JDK proxies.

**Q5. `@within` vs `@target` vs `@annotation`?**
Annotation on the declaring class / on the runtime target class / on the method itself.

**Q6. How do you reuse a pointcut?**
A `@Pointcut` on an empty method, referenced by name (fully qualified across classes); combine with `&&`, `||`, `!`.

**Q7. JDK vs CGLIB proxies? What does Boot use?**
JDK proxies implement interfaces. CGLIB subclasses the class. Boot defaults to CGLIB (`spring.aop.proxy-target-class=true`).

**Q8. Why might an aspect not fire on a method?**
Self-invocation, a private/final method, the object isn't a Spring bean, a pointcut typo or case mismatch, or a proxy-type mismatch (`this` + JDK proxy).

## Quick revision
- `*` = one segment or any type; `..` = any packages or any params.
- `()` zero, `(*)` one, `(..)` any, `(Long,..)` first-is-Long.
- execution / within / @annotation / bean are the big four; this / target / args / @within / @target / @args are advanced.
- JDK proxy = interface only, so `this(ConcreteClass)` fails (verified).
- Centralize named pointcuts; prefer narrow, explicit ones.
