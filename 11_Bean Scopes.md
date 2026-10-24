# Bean Scopes

> **Source:** Lecture 06 (`Notes_Of_Aditya/Lecture 06/code/BeanScopeDemo`), Lecture 07 (`CartService` prototype), Lecture 08 (XML `scope`)
> **Practice:** `Notes_Of_Aditya/Lecture 06/PracticeQuestions.pdf`

## Why this topic matters
Scope decides **how many objects** Spring creates for one bean definition and **how long they live**. Interviewers love the "prototype inside singleton" trap.

---

## 1. Available scopes
| Scope | One object per… | Available in |
|---|---|---|
| **singleton** (default) | Bean definition, per container | All |
| **prototype** | Every `getBean()` / injection request | All |
| request | HTTP request | Web-aware context |
| session | User HTTP session | Web-aware context |
| application | ServletContext (whole web app) | Web-aware context |
| websocket | WebSocket session | Web-aware context |

## 2. Singleton
```java
@Component
@Scope("singleton")          // same as no @Scope at all
public class OrderService {
    public OrderService() { System.out.println("OrderService created"); }
}
```
```java
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
OrderService order  = context.getBean(OrderService.class);
OrderService order2 = context.getBean(OrderService.class);
System.out.println(order == order2);   // true
```
Output:
```
OrderService created      ← printed ONCE, at startup (eager)
true
```

### ⚠️ Spring singleton ≠ GoF Singleton pattern
- GoF Singleton: **one instance per JVM/classloader**, private constructor.
- Spring singleton: **one instance per bean definition per container**. You can still call `new OrderService()` (that object just isn't managed).

```java
@Bean public OrderService getOrder()  { return new OrderService(); }
@Bean public OrderService getOrder2() { return new OrderService(); }
// → 2 bean definitions → 2 singleton objects of the SAME class
// → getBean(OrderService.class) now throws NoUniqueBeanDefinitionException
```

## 3. Prototype
```java
@Component
@Scope("prototype")
public class CartService {
    public CartService() { System.out.println("CartService constructor called"); }
}
```
```java
CartService c1 = context.getBean(CartService.class);
CartService c2 = context.getBean(CartService.class);
System.out.println(c1 == c2);   // false
```
Output:
```
CartService constructor called
CartService constructor called
false
```
- Created **only when requested** (never at startup).
- Spring **does not call destroy callbacks** (`@PreDestroy`) for prototypes. See [Bean Lifecycle](13_Bean%20Lifecycle.md).

XML equivalent: `<bean id="userService" class="in.strikes.UserService" scope="prototype"/>`

## 4. When to use which
| Scope | Best for | Example |
|---|---|---|
| Singleton | **Stateless** beans that only provide behaviour | `PaymentService`, repositories, controllers |
| Prototype | **Stateful** beans holding per-use data | `User`, `OrderRequest`, a cart builder |

```java
// Best candidate for prototype scope — holds state
public class User {
    private String name;
    private int age;
}
```
⚠️ A singleton holding **mutable state** is shared by all threads/requests, which causes thread-safety bugs.

## 5. ⚠️ Prototype injected into a singleton
```java
@Component                      // singleton → created once
public class OrderService {
    private final OrderRequest orderRequest;    // prototype
    public OrderService(OrderRequest orderRequest) { this.orderRequest = orderRequest; }
}
```
The singleton gets **one** prototype instance at creation time and **keeps it forever**.
> Prototype means "new object per request **to the container**", not "new object every time a singleton method runs".

**Fixes to get a fresh instance each time:**
```java
@Component
public class OrderService {
    private final ObjectProvider<OrderRequest> provider;
    public OrderService(ObjectProvider<OrderRequest> provider) { this.provider = provider; }

    public void createOrder() {
        OrderRequest fresh = provider.getObject();   // new prototype every call
    }
}
```
Other options: `jakarta.inject.Provider<T>`, `@Lookup` method injection, `applicationContext.getBean()`, or a scoped proxy: `@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)`.

## 6. Singleton & scope with `A`/`B` from the demo
```java
@Component public class A { public A(OrderService orderService) { ... } }
@Component public class B { public B(OrderService orderService) { ... } }
```
With `OrderService` as a singleton, **A and B share the same instance**. With prototype, **each gets its own**.

---

## Interview Q&A
**Q1. What is the default scope of a Spring bean?**
Singleton.

**Q2. Spring singleton vs Singleton design pattern?**
Spring's is one instance per bean definition per IoC container. The pattern is one instance per classloader, enforced by the class itself.

**Q3. Are Spring singleton beans thread-safe?**
Not automatically. Spring doesn't synchronize them. Keep singletons stateless, or guard shared mutable state.

**Q4. What happens when a prototype bean is injected into a singleton?**
It's injected once, so the singleton always uses the same prototype instance. Use `ObjectProvider`, `@Lookup`, or a scoped proxy to get new instances.

**Q5. Does Spring call `@PreDestroy` on prototype beans?**
No. After handing a prototype out, the container doesn't track it. The client must clean it up.

**Q6. Name the web scopes.**
request, session, application, websocket.

**Q7. When are singleton vs prototype beans created?**
Singletons eagerly at startup (unless `@Lazy`). Prototypes lazily, on each request.

## Quick revision
- Singleton = one per bean definition (default, eager, full lifecycle).
- Prototype = new per request (lazy, no destroy callback).
- Stateless → singleton; stateful → prototype.
- Prototype in a singleton is injected once. Use `ObjectProvider` for fresh instances.
