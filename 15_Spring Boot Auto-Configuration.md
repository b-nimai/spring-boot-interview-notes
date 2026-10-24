# Spring Boot Auto-Configuration and @SpringBootApplication

> **Source:** Lecture 09 (`Notes_Of_Aditya/Lecture 09/code/SpringCoreDemo` + `notes/Notes.pdf`)

## Why this topic matters
"What does `@SpringBootApplication` do?" and "How does auto-configuration work?" are asked in almost every Spring Boot interview.

---

## 1. Spring Core startup vs Spring Boot startup
**Spring Core (manual):**
```java
@Configuration
@ComponentScan("in.strikes")
public class AppConfig {
    @Bean public UserService getUserServiceBean() { return new UserService(); }
}

ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class); // 1. create context
OrderService order = context.getBean(OrderService.class);                            // 2. fetch bean
order.placeOrder();                                                                  // 3. trigger logic
```
**Spring Boot (standard):**
```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```
> Spring Boot **does not replace** Spring Core. Same IoC, same DI. It **starts and configures** the container for you.
> Spring Boot ≠ web API. It works for console, batch, web, and microservices. Web behaviour comes only when you add the web starter.

## 2. `spring-boot-starter-parent` and starters
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.x.x</version>
</parent>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>   <!-- no version: parent manages it -->
</dependency>
```
- **Parent** manages dependency and plugin **versions** (compatible set), Java version, encoding, and build defaults. That avoids version mismatches.
- **Starter** = one dependency that brings a **group** of related libraries transitively.
  - `spring-boot-starter` → core Boot, logging, auto-config (non-web)
  - `spring-boot-starter-web` → Spring MVC, Jackson, embedded Tomcat

## 3. `SpringApplication.run()` does…
1. Creates and prepares the **Environment** (loads properties)
2. Creates the **ApplicationContext** (the right type: web or non-web)
3. Component scanning → registers your beans
4. Applies **auto-configuration**
5. Creates beans and injects dependencies
6. Starts the embedded server (if web)
7. Runs `CommandLineRunner` / `ApplicationRunner`
8. **Returns** a `ConfigurableApplicationContext`

```java
ConfigurableApplicationContext ctx = SpringApplication.run(DemoApplication.class, args);
ctx.getBean(OrderService.class).placeOrder();   // possible, but not the Boot way. Prefer DI + runners.
```

## 4. `@SpringBootApplication` = 3 annotations
```java
@SpringBootConfiguration      // = @Configuration for the main class (also helps tests find config)
@EnableAutoConfiguration      // apply Boot's ready-made configs when conditions match
@ComponentScan                // scan this package + sub-packages
public class DemoApplication { }
```
| Annotation | Purpose |
|---|---|
| `@SpringBootConfiguration` | Marks the main config class, so `@Bean` methods can live in it |
| `@EnableAutoConfiguration` | Turns on auto-configuration |
| `@ComponentScan` | Finds **your** `@Component/@Service/@Repository/@Controller` classes |

### Package structure matters
```
in.strikes
├── DemoApplication.java      ← main class in the ROOT package
├── controller/
├── service/
├── repository/
└── config/
```
Classes **outside** `in.strikes.*` are not scanned. Fix with:
```java
@SpringBootApplication(scanBasePackages = "in.strikes")
```
(The better fix is to keep the main class in the root package.)

## 5. How auto-configuration works
> Auto-configuration = **pre-written configuration classes + condition-based activation**.

```java
@AutoConfiguration
@ConditionalOnClass(PaymentGateway.class)        // 1. Is the library on the classpath?
public class PaymentAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean                    // 2. Has the developer NOT defined one?
    public PaymentGateway paymentGateway() {
        return new DefaultPaymentGateway();
    }
}
```
Three main checks:
1. **Is the class present?** → `@ConditionalOnClass`
2. **Is the bean missing?** → `@ConditionalOnMissingBean` (so **your beans win** over Boot's defaults)
3. **Is the property enabled?** → `@ConditionalOnProperty`

Examples: `DataSourceAutoConfiguration`, `WebMvcAutoConfiguration`, `JacksonAutoConfiguration`.
Boot finds the candidates through `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` inside its JARs.

### Classpath-driven decisions
```
add dependency in pom.xml → Maven downloads JARs → classes on classpath
   → Boot checks conditions → configures matching features
```
- Only `spring-boot-starter`: no web classes, so **no Tomcat**. It runs like a console app and exits.
- Add `spring-boot-starter-web`: web classes are present, so it configures MVC and **starts Tomcat on 8080**, and the app keeps running.

## 6. Startup flow
```
main() → SpringApplication.run()
 → Environment prepared (properties loaded)
 → ApplicationContext created
 → @ComponentScan registers user beans
 → auto-configuration classes evaluated (conditions)
 → missing default beans created
 → dependencies injected
 → context ready
 → CommandLineRunner / ApplicationRunner run
```

## 7. `@ComponentScan` vs `@EnableAutoConfiguration`
| | Finds |
|---|---|
| `@ComponentScan` | **Your** classes |
| `@EnableAutoConfiguration` | **Boot's** ready-made configs |

---

## Debugging tips
- Run with `--debug` (or `debug=true`) to print the **Conditions Evaluation Report** (which auto-configs matched and why).
- Exclude an auto-config: `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)` or `spring.autoconfigure.exclude=...`.

## Interview Q&A
**Q1. What does `@SpringBootApplication` do?**
It combines `@SpringBootConfiguration`, `@EnableAutoConfiguration`, and `@ComponentScan`.

**Q2. How does Spring Boot auto-configuration work?**
Boot ships `@AutoConfiguration` classes listed in `AutoConfiguration.imports`. At startup, each one is evaluated against conditions (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`…) and only matching ones contribute beans.

**Q3. How do you override an auto-configured bean?**
Define your own bean of that type. `@ConditionalOnMissingBean` makes Boot back off.

**Q4. How do you disable a specific auto-configuration?**
`@SpringBootApplication(exclude = X.class)` or `spring.autoconfigure.exclude`.

**Q5. Why should the main class be in the root package?**
Component scanning starts from its package. Classes outside it aren't detected.

**Q6. What is a starter? What is the starter parent?**
A starter is a curated dependency bundle. The parent provides managed versions and build defaults.

**Q7. Why doesn't my Boot app start Tomcat?**
`spring-boot-starter-web` isn't on the classpath, so the web auto-configuration conditions don't match.

**Q8. `@Configuration` vs `@SpringBootConfiguration`?**
Both are config classes. `@SpringBootConfiguration` marks the single main Boot config and is used by `@SpringBootTest` to locate it.

## Quick revision
- Boot = Spring Core + smart startup + auto-configuration.
- `@SpringBootApplication` = Configuration + AutoConfig + ComponentScan.
- Auto-config = conditions on class, bean, and property. Your beans win.
- Classpath decides what's configured. The web starter brings Tomcat.
