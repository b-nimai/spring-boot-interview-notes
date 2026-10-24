# Types of Dependency Injection in Spring

> **Source:** Lecture 05 (`Notes_Of_Aditya/Lecture 05/code/SpringCoreDemo`), Lecture 06 (`CircularDependencyDemo`)
> **Related:** [Inversion of Control and Dependency Injection](05_Inversion%20of%20Control%20and%20Dependency%20Injection.md)

## Why this topic matters
"Constructor vs field vs setter injection, and which do you prefer?" is one of the most common Spring interview questions.

---

## 1. Constructor injection ✅ (preferred)
```java
@Component
public class OrderService {
    private final PaymentService paymentService;   // can be final

    // @Autowired is OPTIONAL when the class has exactly one constructor
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    public void placeOrder() {
        paymentService.pay();
        System.out.println("Order placed");
    }
}
```
**Why preferred**
1. **Mandatory dependency**: the object can't exist without it.
2. **`final` field**: it can't be changed by accident (immutability, thread safety).
3. **Fully initialized** object right after construction.
4. **Testable without Spring**: `new OrderService(new FakePaymentService())`.
5. Constructor-based **circular dependencies fail fast** at startup, which exposes the bad design.

## 2. Field injection ⚠️ (works, not recommended)
```java
@Component
public class OrderService {
    @Autowired
    private PaymentService paymentService;   // injected via reflection after construction
}
```
**Drawbacks**
- The dependency is **hidden**; it's not visible in the constructor.
- **Can't be `final`**.
- **Hard to unit-test** without Spring or reflection.
- The object exists in an **incomplete state** until Spring injects the field.
- Makes it easy to pile up too many dependencies without noticing (a class doing too much).

## 3. Setter injection (optional or changeable dependencies)
```java
@Component
public class OrderService {
    private PaymentService paymentService;

    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```
Flow: Spring calls the **no-arg constructor**, then calls `setPaymentService(...)`.

## 4. How Spring resolves a dependency
```
Need OrderService → constructor needs PaymentService
  → Is there a PaymentService bean?
      0 beans  → NoSuchBeanDefinitionException ("No qualifying bean of type ...")
      1 bean   → inject it
      >1 beans → NoUniqueBeanDefinitionException unless @Primary / @Qualifier
               (or a parameter name matching a bean name)
```

## 5. Comparison table
| | Constructor | Setter | Field |
|---|---|---|---|
| Mandatory deps | ✅ | ❌ | ❌ |
| `final` / immutable | ✅ | ❌ | ❌ |
| Testable without Spring | ✅ | ✅ (call setter) | ❌ |
| Optional deps | ❌ | ✅ | ✅ |
| Circular dependency | Fails at startup | Spring *may* resolve it | Spring *may* resolve it |
| Recommended | **Yes** | For optional deps | Avoid |

---

## Interview Q&A
**Q1. What are the types of DI in Spring?**
Constructor, setter, and field injection.

**Q2. Which one do you prefer and why?**
Constructor injection. Dependencies are explicit and mandatory, fields can be final, the object is always valid, it's easy to test, and circular dependencies show up early.

**Q3. Is `@Autowired` required on a constructor?**
Not if the class has a single constructor (since Spring 4.3). With multiple constructors, mark the one Spring should use.

**Q4. Why is field injection discouraged?**
Hidden dependencies, no immutability, needs reflection or Spring for testing, possible partially-constructed state.

**Q5. When would you use setter injection?**
For optional dependencies or ones you might reconfigure after construction.

**Q6. How do you make an injected dependency optional?**
`@Autowired(required = false)`, `Optional<T>`, `ObjectProvider<T>`, or `@Nullable`.

## Quick revision
- Constructor = required + final + testable → **default choice**.
- Setter = optional. Field = avoid.
- 0 matches → error. More than 1 match → `@Primary` / `@Qualifier`.
