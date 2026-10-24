# Circular Dependency

> **Source:** Lecture 06 (`Notes_Of_Aditya/Lecture 06/code/CircularDependencyDemo`, `BeanInitializationDemo`), Lecture 07 (`BeanLifeCycleDemo/A.java, B.java`)
> **Practice:** `Notes_Of_Aditya/Lecture 06/PracticeQuestions.pdf`

## Why this topic matters
This is a classic interview question: *"What is a circular dependency, why does constructor injection fail, and how do you fix it?"*

---

## 1. Spring creates beans in dependency order
```
OrderController → OrderService → PaymentService → PaymentGateway
Creation order:  PaymentGateway → PaymentService → OrderService → OrderController
```
If the graph has a **cycle** (A → B → A), there's no valid starting point.

## 2. It's a design problem even in plain Java
```java
public class A {
    private B b;
    public A() { System.out.println("A created"); this.b = new B(); }
}
public class B {
    private A a;
    public B() { System.out.println("B created"); this.a = new A(); }
}
// new A() → new B() → new A() → ... → StackOverflowError
```
Spring just **exposes** this problem at startup.

## 3. Constructor injection: fails ❌
```java
@Component
public class OrderService {
    private final PaymentService paymentService;
    public OrderService(PaymentService paymentService) { this.paymentService = paymentService; }
}
@Component
public class PaymentService {
    private final OrderService orderService;
    public PaymentService(OrderService orderService) { this.orderService = orderService; }
}
```
To call `new OrderService(..)` you need a PaymentService, and to call `new PaymentService(..)` you need an OrderService. That's a deadlock, so startup fails with:
```
BeanCurrentlyInCreationException:
Requested bean is currently in creation: Is there an unresolvable circular reference?
```
> Constructor injection is **still recommended**: the failure is a useful signal of bad design.

## 4. Setter / field injection: *may* work
```java
@Component public class OrderService   { @Autowired private PaymentService paymentService; }
@Component public class PaymentService { @Autowired private OrderService orderService; }
```
Creation and injection are separate steps, so Spring can use an **early reference**:
```java
A a = new A();   // A exists, not fully initialized → early reference stored (3-level singleton cache)
B b = new B();
b.setA(a);       // B gets early reference of A
a.setB(b);       // A gets B
```
⚠️ **Spring Boot 2.6+ disables circular references by default**:
```properties
spring.main.allow-circular-references=false   # default
# spring.main.allow-circular-references=true  # possible, but a workaround, NOT a fix
```

## 5. `@Lazy` at the injection point: proxy workaround
From `BeanInitializationDemo`:
```java
@Component
public class OrderService {
    PaymentService paymentService;
    public OrderService(@Lazy PaymentService paymentService) {   // proxy injected, real bean resolved later
        this.paymentService = paymentService;
    }
    public void placeOrder() { paymentService.pay(); System.out.println("Order placed"); }
    public void getOrderDetails() { System.out.println("Order Details"); }
}

@Component @Lazy
public class PaymentService {
    OrderService orderService;
    public PaymentService(OrderService orderService) { this.orderService = orderService; }
    public void pay() { System.out.println("Payment successful"); orderService.getOrderDetails(); }
}
```
Output of `order.placeOrder()`:
```
Payment successful
Order Details
Order placed
```

## 6. `@PostConstruct` workaround (Lecture 07)
```java
@Component
public class A {
    B b;
    public A(B b) { this.b = b; }              // only A → B in constructor
    @PostConstruct
    public void setB() { b.setA(this); }       // wire B → A after A is ready
}
@Component
public class B {
    A a;
    public void setA(A a) { this.a = a; }
}
```

## 7. ✅ Real fix: refactor (the lecture's final code)
```java
@Component
public class OrderService {
    @Autowired private PaymentService paymentService;
    public void placeOrder() { paymentService.pay(); getOrderDetails(); System.out.println("Order placed"); }
    public void getOrderDetails() { System.out.println("Order Details"); }
}
@Component
public class PaymentService {
    public void pay() {
        System.out.println("Payment done");
        // orderService.getOrderDetails();  ← NOT its responsibility → dependency removed
    }
}
```
| Problem | Better design |
|---|---|
| Two services call each other | Move shared logic to a **third service** |
| One service has too many responsibilities | **Split** it |
| Both depend on each other's internals | Interface or **event-based** communication |
| Orchestration mixed with business logic | Separate **orchestrator/coordinator** class |

> A circular dependency is a **design smell**. Responsibilities are mixed.

---

## Interview Q&A
**Q1. What is a circular dependency?**
Two or more beans depend on each other directly or indirectly (A → B → A).

**Q2. Why does constructor injection fail with circular dependencies?**
An object can't be constructed until all its constructor arguments exist. Neither bean can be created first, so Spring throws `BeanCurrentlyInCreationException`.

**Q3. Why can setter or field injection sometimes resolve it?**
The object is instantiated first and dependencies are injected later. Spring exposes an *early reference* of the partially-created singleton (via its three-level singleton cache) to the other bean.

**Q4. Does Spring Boot allow circular references?**
Not by default since 2.6 (`spring.main.allow-circular-references=false`).

**Q5. How do you fix a circular dependency?**
Best: refactor (extract a third class, split responsibilities, use events). Workarounds: `@Lazy` on the injection point (proxy), setter injection, or `@PostConstruct` wiring.

**Q6. Do prototype-scoped circular dependencies work with setter injection?**
No. Early references only work for singletons, so prototype cycles always fail.

## Quick revision
- Cycle → no starting point.
- Constructor injection → `BeanCurrentlyInCreationException` (good, it fails fast).
- Setter/field → early reference (singletons only); Boot disables this by default.
- `@Lazy` = proxy workaround. **Refactor** = real solution.
