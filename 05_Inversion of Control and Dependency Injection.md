# Inversion of Control (IoC) and Dependency Injection (DI)

> **Source:** Lecture 04 (`Notes_Of_Aditya/Lecture 04/code/CoreDemo`)
> **Prerequisite:** [Tight vs Loose Coupling](04_Tight%20vs%20Loose%20Coupling.md)

## Why this topic matters
IoC and DI are **the core idea of Spring**. Almost every Spring interview starts here.

---

## 1. Dependency Injection (DI)
> A class **receives** the objects it depends on **from outside**, instead of creating them itself.

"Don't create your dependency, receive your dependency."
DI is a **design principle that also works in plain Java**. Spring did not invent DI; **Spring automates it**.

```java
public class OrderService {
    private NotificationService notificationService;

    public OrderService(NotificationService notificationService) { // injected from outside
        this.notificationService = notificationService;
    }

    public void placeOrder() {
        System.out.println("Order placed successfully");
        notificationService.sendNotification();
    }
}

// Main acts as a tiny "container": it creates and wires objects
NotificationService ns = new EmailService();
OrderService orderService = new OrderService(ns);
orderService.placeOrder();
```

## 2. Types of DI
| Type | How | When |
|---|---|---|
| **Constructor injection** | Dependency passed in the constructor | **Required** dependencies (preferred) |
| **Setter injection** | `setX(dep)` after creation | **Optional** or changeable dependencies |
| **Field injection** | `@Autowired` on a field (Spring only) | Works, but not recommended |

```java
// Setter injection (plain Java)
OrderService order = new OrderService();
order.setNotification(new EmailService());
```

## 3. Inversion of Control (IoC)
Ask: **who controls creation of `EmailService`?**
```
Before DI:                      After DI:
OrderService                    Main (or Spring)
   │ creates                       │ creates
   ▼                               ▼
EmailService                    EmailService ──provided to──► OrderService
```
- Before: the class controlled creating what it needs.
- After: something **outside** controls it. The control is **inverted**.

### IoC vs DI
| IoC | DI |
|---|---|
| **Principle**: give control of object creation and wiring to an external system | **Technique** to achieve IoC: pass dependencies in from outside |

(Other ways to achieve IoC include Service Locator, the Template Method pattern, and events/callbacks.)

## 4. Where Spring fits
In a big app, `Main` would end up creating and wiring hundreds of objects. **Spring IoC Container** takes over that job:
1. **Creates** objects
2. **Manages** objects (scope, lifecycle)
3. **Connects** objects (injects dependencies)

```
Plain Java:  Main → creates EmailService → creates OrderService → injects
Spring:      IoC Container → creates EmailService bean → creates OrderService bean + injects
```

## 5. What is a Bean?
> A **bean** is an object **created and managed by the Spring IoC container**.

Every bean is an object, but not every object is a bean. `new EmailService()` written by you is just a normal object.

---

## Interview Q&A
**Q1. What is Dependency Injection?**
A design technique where an object's dependencies are supplied from outside (constructor, setter, or field) instead of being created inside the object.

**Q2. What is IoC? How is it different from DI?**
IoC is the principle that control of object creation and wiring moves out of the class to an external container. DI is the most common way to implement IoC.

**Q3. Which DI type is preferred and why?**
Constructor injection: dependencies are mandatory and explicit, fields can be `final` (immutable), the object is never half-initialized, and it's easy to unit-test without Spring.

**Q4. When would you use setter injection?**
For optional dependencies, or ones that may be replaced after construction.

**Q5. Is DI specific to Spring?**
No. It's a general design principle. Spring (and Guice, Dagger, CDI) automate it.

**Q6. What is a Spring Bean?**
An object whose creation, dependency wiring, and lifecycle are managed by the Spring IoC container.

**Q7. What are the benefits of DI?**
Loose coupling, easier testing (inject fakes), easier switching of implementations, and reusable classes.

## Quick revision
- DI = receive dependencies; IoC = control moved outside the class.
- IoC is the principle, DI is the technique.
- Spring IoC Container = creates, manages, and wires objects (beans).
- Prefer constructor injection.
