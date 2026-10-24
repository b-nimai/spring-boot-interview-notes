# CommandLineRunner and ApplicationRunner

> **Source:** Lecture 10 (`Notes_Of_Aditya/Lecture 10/code/demo/DemoRunner.java`), Lecture 09 notes

## Why this topic matters
In a non-web Boot app there's no HTTP request to trigger your code. Runners are the Boot way to **run logic once the context is ready**. They're also used for seeding data, warm-up, and startup checks.

---

## 1. Why not call beans from `main()`?
```java
public static void main(String[] args) {
    ConfigurableApplicationContext context = SpringApplication.run(DemoApplication.class, args);
    PaymentGateway pg = context.getBean(PaymentGateway.class);   // works, but it's the manual style again
    pg.print();
}
```
That's back to "start the context manually → fetch the bean manually → call the method manually".
**Better:** let Spring create a bean that runs automatically after startup.

## 2. `CommandLineRunner`
```java
@Component
public class DemoRunner implements CommandLineRunner {

    private final PaymentGateway paymentGateway;

    public DemoRunner(PaymentGateway paymentGateway) {   // DI works as usual
        this.paymentGateway = paymentGateway;
    }

    @Override
    public void run(String... args) throws Exception {   // raw String args
        paymentGateway.print();
    }
}
```
Flow:
```
Spring creates DemoRunner bean → injects PaymentGateway
  → context ready → Boot detects CommandLineRunner beans → calls run()
```
Reading args:
```bash
java -jar app.jar hello world
```
```java
@Override
public void run(String... args) {
    for (String arg : args) System.out.println(arg);   // hello, world
}
```

## 3. `ApplicationRunner`
```java
@Component
public class AppStartupRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        System.out.println(args.getOptionNames());           // [provider, retry]
        System.out.println(args.getOptionValues("provider")); // [Razorpay]
        System.out.println(args.getNonOptionArgs());          // plain args
    }
}
```
```bash
java -jar app.jar --provider=Razorpay --retry=3
```

## 4. Difference
| | `CommandLineRunner` | `ApplicationRunner` |
|---|---|---|
| Method | `run(String... args)` | `run(ApplicationArguments args)` |
| Args | Raw strings | Parsed: option (`--k=v`) vs non-option |
| Use when | Simple args / just run code | You need named options |

Both run **after** the ApplicationContext is fully started, before `SpringApplication.run()` returns.

## 5. Multiple runners: ordering
```java
@Component @Order(1) class FirstRunner  implements CommandLineRunner { ... }
@Component @Order(2) class SecondRunner implements ApplicationRunner { ... }
```
Lower `@Order` value runs first (or implement `Ordered`).

## 6. Where runners sit in the startup flow
```
SpringApplication.run()
 → Environment + properties loaded
 → context created, beans scanned + auto-configured + injected
 → @Value / @ConfigurationProperties bound
 → embedded server started (if web)
 → CommandLineRunner / ApplicationRunner.run()
 → app exits (non-web) or keeps serving (web)
```

---

## Common pitfalls
- An exception thrown in `run()` **fails application startup**.
- Long-running work in a runner blocks startup completion. Use async or a scheduler instead.
- A runner without `@Component` (not a bean) never runs.

## Interview Q&A
**Q1. What is CommandLineRunner?**
A Spring Boot callback interface whose `run(String... args)` executes once, after the application context has started.

**Q2. CommandLineRunner vs ApplicationRunner?**
Same timing. CommandLineRunner gets raw `String[]` args; ApplicationRunner gets parsed `ApplicationArguments` (option vs non-option).

**Q3. How do you control the order of multiple runners?**
`@Order` or the `Ordered` interface.

**Q4. Typical use cases?**
Seeding a DB, cache warm-up, printing config, validating external connectivity, one-off batch jobs in console apps.

**Q5. `@PostConstruct` vs CommandLineRunner?**
`@PostConstruct` runs during **one bean's** initialization, when other beans may not be ready yet. A runner runs after the **whole context** is ready.

## Quick revision
- Runners = run code after startup without an HTTP request.
- CommandLineRunner → `String...`; ApplicationRunner → `ApplicationArguments`.
- Order with `@Order`. An exception in `run()` means startup fails.
