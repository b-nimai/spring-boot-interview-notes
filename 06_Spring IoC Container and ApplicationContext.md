# Spring IoC Container and ApplicationContext

> **Source:** Lecture 05 (`Notes_Of_Aditya/Lecture 05/code/SpringCoreDemo`)
> **Related:** [Types of Dependency Injection in Spring](07_Types%20of%20Dependency%20Injection%20in%20Spring.md) · [Component vs Bean](09_Component%20vs%20Bean.md) · [Primary and Qualifier](08_Resolving%20Multiple%20Beans%20-%20Primary%20and%20Qualifier.md)

## Why this topic matters
This is how pure Spring Core starts up, without Spring Boot. Once you know this flow, `@SpringBootApplication` is easy to understand.

---

## 1. Setup
Create a Maven project and add **`spring-context`**. It gives you ApplicationContext, annotation config, component scanning, and DI.
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>7.0.7</version>
</dependency>
```

## 2. Two ways to give Spring configuration
1. **Annotation-based** (modern): `@Component`, `@Configuration`, `@ComponentScan`, `@Autowired`, `@Bean`
2. **XML-based** (legacy): see [XML Based Configuration](14_XML%20Based%20Configuration.md)

## 3. Core annotations and classes
| Item | Meaning |
|---|---|
| `@Component` | "This class is **eligible** to become a bean." |
| `@ComponentScan("pkg")` | "Search **here** (package + sub-packages) for components." No value means the package of the config class. |
| `@Configuration` | "This class is a **source of bean definitions**" (may contain `@Bean` methods). It is itself a `@Component`. |
| `ApplicationContext` | **Interface** that represents the IoC container. You can't do `new ApplicationContext()`. |
| `AnnotationConfigApplicationContext` | Implementation that boots the container from annotation config. |
| `context.getBean(X.class)` | "Spring, give me the bean of type X." |

Stereotypes that are also detected by scanning: `@Service`, `@Repository`, `@Controller` (all specializations of `@Component`).

### Why `AppConfig.class`?
`AppConfig.class` is not an object. It's a `Class` object holding **metadata** (annotations, methods). Spring reads it with **reflection** to learn its startup instructions.

## 4. Code from the lecture
```java
@Configuration
@ComponentScan("in.coderarmy")
public class AppConfig {

    @Bean
    public User createUser() {                 // custom object creation: Spring can't guess "Aditya", 28
        return new User("Aditya", 28);
    }

    @Bean
    public CartService createCartService() {   // class from another module/JAR → can't add @Component
        return new CartService();
    }
}
```
```java
public class Main {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

        OrderService order = context.getBean(OrderService.class);
        order.placeOrder();
    }
}
```

## 5. What happens internally at startup
```
new AnnotationConfigApplicationContext(AppConfig.class)
 1. Start IoC container (ApplicationContext)
 2. Read AppConfig.class (reflection)
 3. Process @Configuration
 4. Process @ComponentScan → scan package + sub-packages
 5. Find @Component/@Service/@Repository/@Controller classes + @Bean methods
 6. Create BeanDefinitions (metadata: name, class, scope, deps, lazy?, init/destroy)
 7. Resolve dependency order
 8. Instantiate beans (dependencies first)
 9. Inject dependencies
10. Store singletons in the container
11. Return them on getBean()
```
**BeanDefinition** = a *recipe*, not the object:
```
Bean name    -> paymentService
Bean class   -> in.coderarmy.PaymentService
Scope        -> singleton
Dependencies -> none
```
Spring builds the full list of definitions first, so it knows the **whole object graph** (dependencies, scopes, lazy/eager) before creating anything.

## 6. Errors you will see
| Error | Cause |
|---|---|
| `NoSuchBeanDefinitionException` / *No qualifying bean of type 'X' available* | Class not annotated, package not scanned, or no `@Bean` method |
| `NoUniqueBeanDefinitionException` | More than one bean of that type → use `@Primary` / `@Qualifier` |

## 7. BeanFactory vs ApplicationContext
| BeanFactory | ApplicationContext |
|---|---|
| Basic container interface | Extends BeanFactory |
| Lazy by default | Pre-instantiates singletons eagerly |
| Just DI | + events, i18n (MessageSource), AOP integration, environment/properties |

In real apps you use **ApplicationContext**.

## 8. Why not create everything in `main()`?
That makes `main()` a manual container again. Keep `main()` for **starting** the app and let Spring handle creation, wiring, and lifecycle.

---

## Interview Q&A
**Q1. What is the Spring IoC container?**
The component (ApplicationContext) that reads configuration metadata, creates beans, injects dependencies, and manages their lifecycle.

**Q2. `@Component` vs `@ComponentScan`?**
`@Component` marks a class as a candidate bean. `@ComponentScan` tells Spring where to look for such classes.

**Q3. What if `@ComponentScan` has no package?**
It scans the package of the annotated config class and its sub-packages.

**Q4. What is a BeanDefinition?**
Metadata describing how to create a bean (class, name, scope, dependencies, lazy flag, init/destroy methods). It is created before the actual object.

**Q5. BeanFactory vs ApplicationContext?**
ApplicationContext extends BeanFactory with eager singleton creation, event publishing, i18n, environment abstraction, and AOP integration.

**Q6. Is `@Configuration` a bean?**
Yes. It's meta-annotated with `@Component`, so it's scanned and registered (as a CGLIB-proxied bean).

**Q7. Name implementations of ApplicationContext.**
`AnnotationConfigApplicationContext`, `ClassPathXmlApplicationContext`, `FileSystemXmlApplicationContext`, and the web contexts Spring Boot uses (e.g. `AnnotationConfigServletWebServerApplicationContext`).

## Quick revision
- Add `spring-context` → `@Configuration` + `@ComponentScan` → `new AnnotationConfigApplicationContext(AppConfig.class)` → `getBean()`.
- Spring builds BeanDefinitions first, then objects.
- No bean → NoSuchBeanDefinition; two beans → NoUniqueBeanDefinition.
