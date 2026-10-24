# @Component vs @Bean

> **Source:** Lecture 05 (`Notes_Of_Aditya/Lecture 05/code/SpringCoreDemo/AppConfig.java`), Lecture 06 (`BeanScopeDemo/AppConfig.java`)

## Why this topic matters
Both register beans, but they're used in different situations. Interviewers like asking when you **can't** use `@Component`.

---

## 1. `@Component`: automatic detection (class level)
```java
@Component
public class CardPayment implements PaymentService {
    public void pay() { System.out.println("Paying via card"); }
}
```
Spring finds it through `@ComponentScan`. The bean name is `cardPayment`.

## 2. `@Bean`: manual registration (method level)
Used inside a `@Configuration` class. Spring **calls the method** and registers the **returned object** as a bean.
```java
@Configuration
public class AppConfig {
    @Bean
    public PaymentService paymentService() {        // bean name = "paymentService" (method name)
        return new PaymentService();
    }

    @Bean("myPaymentService")                       // custom name
    public PaymentService anotherPaymentService() { return new PaymentService(); }
}
```

### When you need `@Bean`
1. **Third-party / library classes**: you can't open the class to add `@Component`.
   In the lecture, `CartService` comes from another Maven module (`SpringCoreDemo2`), so:
   ```java
   @Bean
   public CartService createCartService() { return new CartService(); }
   ```
2. **Custom construction logic**: Spring can't guess constructor values.
   ```java
   @Bean
   public User createUser() { return new User("Aditya", 28); }
   ```
3. Creating **several configured instances** of the same class (e.g. two `DataSource`s).

### `@Bean` with dependencies
```java
@Bean
public PaymentService paymentService() { return new PaymentService(); }

@Bean
public OrderService orderService(PaymentService paymentService) {   // ✅ Spring injects it
    return new OrderService(paymentService);
}

// Alternative: call the method directly
@Bean
public OrderService orderService2() {
    return new OrderService(paymentService());   // still returns the SAME singleton
}
```
**Why is it the same singleton?** `@Configuration` classes are **CGLIB-proxied**. The proxy intercepts `paymentService()` and returns the existing bean from the container. With `@Configuration(proxyBeanMethods = false)`, or `@Bean` methods inside a plain `@Component` ("lite mode"), each call would create a **new** object. Prefer method parameters.

## 3. Comparison
| Point | `@Component` | `@Bean` |
|---|---|---|
| Where | On a **class** | On a **method** (in `@Configuration`) |
| Style | Automatic detection | Manual registration |
| Needs component scan? | Yes | No (but the config class must be loaded) |
| Default bean name | Class name (camelCase) | Method name |
| Best for | Your own classes | Library classes, custom creation logic |
| Control over creation | Spring calls the constructor | You write the creation code |

## 4. Singleton is per bean definition, not per class
```java
@Bean public OrderService getOrder()  { return new OrderService(); }
@Bean public OrderService getOrder2() { return new OrderService(); }
```
This gives **two bean definitions**, so **two singleton objects** of the same class. `context.getBean(OrderService.class)` now throws `NoUniqueBeanDefinitionException`.

## 5. ⚠️ Don't register the same class both ways
`@Component` on `PaymentService` **and** a `@Bean PaymentService paymentService()`:
- Different names → two beans → by-type injection becomes ambiguous.
- Same name → bean overriding rules apply (**Spring Boot disables overriding by default**, so startup fails).

Pick one approach.

## 6. Stereotype annotations
`@Service`, `@Repository`, `@Controller` / `@RestController` are all `@Component` with extra meaning:
- `@Repository` also translates persistence exceptions into Spring's `DataAccessException`.
- `@Controller` is picked up by Spring MVC for request mappings.

---

## Interview Q&A
**Q1. Difference between `@Component` and `@Bean`?**
`@Component` is class-level and detected automatically by component scanning. `@Bean` is method-level inside a config class, and you write the creation code yourself.

**Q2. When must you use `@Bean`?**
For classes you can't annotate (third-party libraries such as `RestTemplate` or `ObjectMapper`), or when creation needs custom logic or values.

**Q3. What is the default name of a `@Bean`?**
The method name, unless you set `@Bean("name")`.

**Q4. Calling one `@Bean` method from another: new object or the same?**
The same singleton in a `@Configuration` class (CGLIB proxy). A new object in lite mode (`proxyBeanMethods = false` or a `@Component` class).

**Q5. `@Component` vs `@Service` vs `@Repository`?**
They're all components. `@Service` and `@Repository` add semantic meaning, and `@Repository` adds exception translation.

**Q6. Two `@Bean` methods return the same type. How many objects?**
Two, because singleton scope is per bean definition.

## Quick revision
- `@Component` = "find me". `@Bean` = "here's how to build me".
- `@Bean` is for library classes and custom construction.
- `@Configuration` proxying keeps inter-`@Bean` calls singleton.
- Don't register the same class with both.
