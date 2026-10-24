# Bean Lifecycle

> **Source:** Lecture 07 (`Notes_Of_Aditya/Lecture 07/code/BeanLifeCycleDemo`), Lecture 08 (XML `init-method` / `destroy-method`)

## Why this topic matters
"Explain the Spring bean lifecycle" is a top Spring Core interview question. Know the **order** of the phases and the **three ways** to hook into init and destroy.

---

## 1. The full lifecycle
```
Container starts
  ↓ read config / annotations (reflection on AppConfig.class)
  ↓ create BeanDefinition          (metadata only)
  ↓ instantiate                    (constructor)
  ↓ inject dependencies            (constructor / setter / field)
  ↓ Aware callbacks                (BeanNameAware → BeanFactoryAware → ApplicationContextAware)
  ↓ BeanPostProcessor.postProcessBeforeInitialization
  ↓ init callbacks                 (@PostConstruct → afterPropertiesSet() → initMethod)
  ↓ BeanPostProcessor.postProcessAfterInitialization   (AOP proxies are created here)
  ↓ bean READY → application uses it
  ↓ context.close()
  ↓ destroy callbacks              (@PreDestroy → destroy() → destroyMethod)
Bean removed
```
> **Instantiation ≠ initialization.** Instantiation = the object is created. Initialization = the object is prepared after its dependencies are injected.

## 2. Aware interfaces: the container tells the bean about itself
```java
public class UserService implements BeanNameAware, ApplicationContextAware {

    public UserService() { System.out.println("UserService constructor called"); }

    @Override
    public void setBeanName(String name) {                       // Spring → bean
        System.out.println("Bean name is " + name);
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) throws BeansException {
        System.out.println("ApplicationContext name is " + ctx.getClass());
    }
}
```
Output order:
```
UserService constructor called
Bean name is userService
ApplicationContext name is class ...AnnotationConfigApplicationContext
```
- The method is `setBeanName` because the **direction is container → bean**.
- Calling `userService.setBeanName("x")` yourself is just a normal method call. It does **not** rename the bean in the container.
- Rarely used in business code. Useful for framework or infrastructure code.

## 3. Initialization: three ways
Use it for: validating config, warming a cache, filling a map, opening resources.

```java
// Option 1 ✅ @PostConstruct  (jakarta.annotation.PostConstruct)
@PostConstruct
public void start2() {
    System.out.println("Bean is Ready");
    mp.put(1, "Aditya");
    mp.put(2, "Rohit");
}

// Option 2: InitializingBean (couples the class to Spring)
public class CartService implements InitializingBean {
    @Override public void afterPropertiesSet() { ... }
}

// Option 3: custom init method (keeps the class Spring-free; good for library classes)
@Bean(initMethod = "start", destroyMethod = "stop")
public CartService getCartBean() { return new CartService(); }
```
**Order if all three are used:** `@PostConstruct` → `afterPropertiesSet()` → `initMethod`.

> For plain Spring Core (no Boot) you may need the `jakarta.annotation:jakarta.annotation-api` dependency for `@PostConstruct` / `@PreDestroy`.

## 4. Destruction: three ways
Use it for: closing DB connections, stopping threads, flushing data, clearing caches.
```java
@PreDestroy                                   // Option 1 ✅
public void stop() {
    mp.clear();
    System.out.println("Bean is getting destroyed");
}
// Option 2: implements DisposableBean → destroy()
// Option 3: @Bean(destroyMethod = "stop")   /  XML destroy-method="cleanup"
```
**Order:** `@PreDestroy` → `DisposableBean.destroy()` → `destroyMethod`.
Destroy callbacks run only when the context is closed, so use a `ConfigurableApplicationContext`:
```java
ConfigurableApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
...
context.close();          // or try-with-resources, or context.registerShutdownHook()
```

## 5. Complete demo class (from the lecture)
```java
@Component
@Scope("prototype")
public class CartService implements BeanNameAware, ApplicationContextAware {
    Map<Integer, String> mp;

    public CartService() { mp = new HashMap<>(); System.out.println("CartService constructor called"); }

    @Override public void setBeanName(String name) { System.out.println("Bean name is " + name); }
    @Override public void setApplicationContext(ApplicationContext c) { System.out.println("ApplicationContext name is " + c.getClass()); }

    @PostConstruct
    public void start2() { System.out.println("Bean is Ready"); mp.put(1, "Aditya"); mp.put(2, "Rohit"); }

    public String getValue(int key) { return mp.get(key); }

    @PreDestroy
    public void stop() { mp.clear(); System.out.println("Bean is getting destroyed"); }
}
```
```java
CartService cart = context.getBean(CartService.class);
System.out.println(cart.getValue(1));
context.close();
```
Output (prototype):
```
CartService constructor called
Bean name is cartService
ApplicationContext name is class org.springframework.context.annotation.AnnotationConfigApplicationContext
Bean is Ready
Aditya
                         ← NO "Bean is getting destroyed" (prototype!)
```
With singleton scope, the first four lines print at **startup**, and `Bean is getting destroyed` prints on `close()`.

## 6. Singleton vs prototype lifecycle
| | Singleton | Prototype |
|---|---|---|
| Created | At startup (eager) | On each request |
| Init callbacks | ✅ | ✅ |
| Stored in container | ✅ (singleton cache) | ❌ |
| Destroy callbacks on `close()` | ✅ | ❌ |

**Why no destroy for prototypes?** Spring would have to keep a reference to every prototype it ever created. That prevents garbage collection and causes **memory leaks**. The caller owns the prototype and must clean up (try-with-resources, manual close).
This applies even to a prototype injected into a singleton: its `@PreDestroy` won't run.

## 7. `@Lazy` and the lifecycle
For a `@Lazy` singleton, the BeanDefinition is registered at startup, but instantiation, DI, Aware callbacks, and init happen only when it's first needed.

---

## Interview Q&A
**Q1. Explain the Spring bean lifecycle.**
BeanDefinition → instantiate → populate dependencies → Aware callbacks → BeanPostProcessor (before) → init (`@PostConstruct`, `afterPropertiesSet`, init-method) → BeanPostProcessor (after) → ready → on shutdown, destroy (`@PreDestroy`, `destroy()`, destroy-method).

**Q2. What are the ways to run code after a bean is initialized?**
`@PostConstruct`, `InitializingBean.afterPropertiesSet()`, `@Bean(initMethod=...)` / XML `init-method`. (Also `BeanPostProcessor` for cross-cutting logic.)

**Q3. Order if all init mechanisms are present?**
`@PostConstruct` → `afterPropertiesSet()` → custom init method. Destroy follows the same pattern.

**Q4. Why prefer `@PostConstruct` over the constructor for setup logic?**
In the constructor, field/setter-injected dependencies aren't set yet. `@PostConstruct` runs after all injection is done.

**Q5. Is `@PreDestroy` called for prototype beans?**
No. The container doesn't track prototype instances after handing them out.

**Q6. What is a BeanPostProcessor?**
A hook that lets you modify or wrap every bean before and after initialization. Spring uses it for `@Autowired`, `@PostConstruct`, and AOP proxies.

**Q7. What are Aware interfaces?**
Callback interfaces (`BeanNameAware`, `BeanFactoryAware`, `ApplicationContextAware`) through which the container gives a bean information about itself.

**Q8. Why does `@PreDestroy` not run when my `main()` ends?**
The context was never closed. Call `context.close()` or `registerShutdownHook()`. Spring Boot registers a shutdown hook automatically.

## Quick revision
- Order: definition → constructor → DI → Aware → BPP-before → init → BPP-after → ready → destroy.
- Init: `@PostConstruct` > `afterPropertiesSet` > initMethod (same order for destroy).
- Destroy needs `context.close()`.
- Prototype: init yes, destroy **no**.
