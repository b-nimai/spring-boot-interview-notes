# Spring Boot Notes (Topic-wise)

Topic-wise notes built from the Coder Army Spring course (`../Notes_Of_Aditya/`), meant for **understanding concepts** and **interview preparation**.
Each note has: why it matters → concept → key code (from the lectures) → pitfalls → **Interview Q&A** → quick revision.

## Study order (Lectures 1–40, complete course)

### Part 1: Spring Core and Boot basics (Lectures 1–10)

| # | Topic | Lecture |
|---|---|---|
| 1 | [Introduction to Spring and Spring Boot](01_Introduction%20to%20Spring%20and%20Spring%20Boot.md) | 01 |
| 2 | [First Spring Boot Application](02_First%20Spring%20Boot%20Application.md) | 02 |
| 3 | [Maven](03_Maven.md) | 03 |
| 4 | [Tight vs Loose Coupling](04_Tight%20vs%20Loose%20Coupling.md) | 04 |
| 5 | [Inversion of Control and Dependency Injection](05_Inversion%20of%20Control%20and%20Dependency%20Injection.md) | 04 |
| 6 | [Spring IoC Container and ApplicationContext](06_Spring%20IoC%20Container%20and%20ApplicationContext.md) | 05 |
| 7 | [Types of Dependency Injection in Spring](07_Types%20of%20Dependency%20Injection%20in%20Spring.md) | 05 |
| 8 | [Resolving Multiple Beans - Primary and Qualifier](08_Resolving%20Multiple%20Beans%20-%20Primary%20and%20Qualifier.md) | 05 |
| 9 | [Component vs Bean](09_Component%20vs%20Bean.md) | 05 |
| 10 | [Circular Dependency](10_Circular%20Dependency.md) | 06 |
| 11 | [Bean Scopes](11_Bean%20Scopes.md) | 06 |
| 12 | [Eager vs Lazy Initialization](12_Eager%20vs%20Lazy%20Initialization.md) | 06 |
| 13 | [Bean Lifecycle](13_Bean%20Lifecycle.md) | 07 |
| 14 | [XML Based Configuration](14_XML%20Based%20Configuration.md) | 08 |
| 15 | [Spring Boot Auto-Configuration](15_Spring%20Boot%20Auto-Configuration.md) | 09 |
| 16 | [Externalized Configuration - application.properties, Value and ConfigurationProperties](16_Externalized%20Configuration%20-%20application.properties%2C%20Value%20and%20ConfigurationProperties.md) | 10 |
| 17 | [CommandLineRunner and ApplicationRunner](17_CommandLineRunner%20and%20ApplicationRunner.md) | 10 |

### Part 2: Web, data and REST APIs (Lectures 11–20)

| # | Topic | Lecture |
|---|---|---|
| 18 | [Layered Architecture and REST CRUD API](18_Layered%20Architecture%20and%20REST%20CRUD%20API.md) | 11, 12 |
| 19 | [Spring Data JPA - Entity, Repository and Database Config](19_Spring%20Data%20JPA%20-%20Entity%2C%20Repository%20and%20Database%20Config.md) | 12 |
| 20 | [Soft Delete and Derived Query Methods](20_Soft%20Delete%20and%20Derived%20Query%20Methods.md) | 13 |
| 21 | [Servlets, Tomcat and Servlet Lifecycle](21_Servlets%2C%20Tomcat%20and%20Servlet%20Lifecycle.md) | 14 |
| 22 | [Spring MVC Architecture and DispatcherServlet](22_Spring%20MVC%20Architecture%20and%20DispatcherServlet.md) | 15 |
| 23 | [Spring MVC with JSP and ViewResolver](23_Spring%20MVC%20with%20JSP%20and%20ViewResolver.md) | 15 |
| 24 | [DTOs and Bean Validation](24_DTOs%20and%20Bean%20Validation.md) | 16 |
| 25 | [Exception Handling and Global Exception Handler](25_Exception%20Handling%20and%20Global%20Exception%20Handler.md) | 17 |
| 26 | [YAML Configuration and Spring Profiles](26_YAML%20Configuration%20and%20Spring%20Profiles.md) | 18 |
| 27 | [Servlet Filters - Basics, FilterChain and Order](27_Servlet%20Filters%20-%20Basics%2C%20FilterChain%20and%20Order.md) | 19 |
| 28 | [Advanced Filters - Wrappers, FilterRegistrationBean and OncePerRequestFilter](28_Advanced%20Filters%20-%20Wrappers%2C%20FilterRegistrationBean%20and%20OncePerRequestFilter.md) | 20 |

### Part 3: Interceptors, AOP and data access (Lectures 21–30)

| # | Topic | Lecture |
|---|---|---|
| 29 | [Spring MVC Interceptors](29_Spring%20MVC%20Interceptors.md) | 21 |
| 30 | [Introduction to AOP and Cross-Cutting Concerns](30_Introduction%20to%20AOP%20and%20Cross-Cutting%20Concerns.md) | 22 |
| 31 | [AOP Advice Types](31_AOP%20Advice%20Types.md) | 23 |
| 32 | [AOP Pointcut Expressions](32_AOP%20Pointcut%20Expressions.md) | 24 |
| 33 | [AOP Pointcut Practice Questions](33_AOP%20Pointcut%20Practice%20Questions.md) | 24 (practice) |
| 34 | [Custom Annotations and AOP Proxy Internals](34_Custom%20Annotations%20and%20AOP%20Proxy%20Internals.md) | 25 |
| 35 | [JDBC Fundamentals](35_JDBC%20Fundamentals.md) | 26 |
| 36 | [Spring JDBC - DataSource, HikariCP and JdbcTemplate](36_Spring%20JDBC%20-%20DataSource%2C%20HikariCP%20and%20JdbcTemplate.md) | 27 |
| 37 | [Hibernate and JPA Entity Mapping](37_Hibernate%20and%20JPA%20Entity%20Mapping.md) | 28 |
| 38 | [Hibernate Internals - Persistence Context, Dirty Checking and Flush](38_Hibernate%20Internals%20-%20Persistence%20Context%2C%20Dirty%20Checking%20and%20Flush.md) | 29 |
| 39 | [JPA Relationships](39_JPA%20Relationships.md) | 30 |

### Part 4: JPA performance, transactions, security and testing (Lectures 31–40)

| # | Topic | Lecture |
|---|---|---|
| 40 | [JPA Cascading, Lazy Loading and the N+1 Problem](40_JPA%20Cascading%2C%20Lazy%20Loading%20and%20the%20N%2B1%20Problem.md) | 31 |
| 41 | [Spring Data JPA - Repositories, Queries, Sorting and Pagination](41_Spring%20Data%20JPA%20-%20Repositories%2C%20Queries%2C%20Sorting%20and%20Pagination.md) | 32 |
| 42 | [Transactions and ACID](42_Transactions%20and%20ACID.md) | 33 |
| 43 | [Transaction Propagation and Isolation](43_Transaction%20Propagation%20and%20Isolation.md) | 34 |
| 44 | [Spring Security Fundamentals](44_Spring%20Security%20Fundamentals.md) | 35 |
| 45 | [Database Authentication and Password Hashing](45_Database%20Authentication%20and%20Password%20Hashing.md) | 36 |
| 46 | [Spring Security Architecture and Login Flow](46_Spring%20Security%20Architecture%20and%20Login%20Flow.md) | 37 |
| 47 | [JWT Authentication](47_JWT%20Authentication.md) | 38 |
| 48 | [OAuth2 and OpenID Connect - Google Login](48_OAuth2%20and%20OpenID%20Connect%20-%20Google%20Login.md) | 39 |
| 49 | [Testing with JUnit, Mockito and MockMvc](49_Testing%20with%20JUnit%2C%20Mockito%20and%20MockMvc.md) | 40 |

### Part 5: Interview extras (beyond the course)

| # | Topic | Why |
|---|---|---|
| 50 | [NestJS to Spring Boot Mapping](50_NestJS%20to%20Spring%20Boot%20Mapping.md) | Side-by-side NestJS ↔ Spring code for every concept, plus comparison questions |

### Part 6: Core Java for interviews

| # | Topic |
|---|---|
| 51 | [Java Fundamentals - Types, Strings and Memory Basics](51_Java%20Fundamentals%20-%20Types%2C%20Strings%20and%20Memory%20Basics.md) |
| 52 | [OOP in Java - Classes, Interfaces, Inheritance and Polymorphism](52_OOP%20in%20Java%20-%20Classes%2C%20Interfaces%2C%20Inheritance%20and%20Polymorphism.md) |
| 53 | [equals, hashCode, Immutability and Records](53_equals%2C%20hashCode%2C%20Immutability%20and%20Records.md) |
| 54 | [Collections Framework](54_Collections%20Framework.md) |
| 55 | [HashMap Internals](55_HashMap%20Internals.md) |
| 56 | [Generics](56_Generics.md) |
| 57 | [Exceptions](57_Exceptions.md) |
| 58 | [Java 8+ - Lambdas, Functional Interfaces, Streams and Optional](58_Java%208%2B%20-%20Lambdas%2C%20Functional%20Interfaces%2C%20Streams%20and%20Optional.md) |
| 59 | [Multithreading Basics - Threads, Executors and CompletableFuture](59_Multithreading%20Basics%20-%20Threads%2C%20Executors%20and%20CompletableFuture.md) |
| 60 | [Concurrency - Synchronization, Locks, volatile and Atomics](60_Concurrency%20-%20Synchronization%2C%20Locks%2C%20volatile%20and%20Atomics.md) |
| 61 | [JVM Memory, Class Loading and Garbage Collection](61_JVM%20Memory%2C%20Class%20Loading%20and%20Garbage%20Collection.md) |
| 62 | [Modern Java 9-21 Features](62_Modern%20Java%209-21%20Features.md) |

### Part 7: Spring gap topics for SDE-1/2 (beyond the course)

| # | Topic |
|---|---|
| 63 | [Caching with Spring Cache and Redis](63_Caching%20with%20Spring%20Cache%20and%20Redis.md) |
| 64 | [Async and Scheduled Tasks](64_Async%20and%20Scheduled%20Tasks.md) |
| 65 | [REST Clients - RestClient, WebClient and OpenFeign](65_REST%20Clients%20-%20RestClient%2C%20WebClient%20and%20OpenFeign.md) |
| 66 | [Actuator, Logging and Observability](66_Actuator%2C%20Logging%20and%20Observability.md) |
| 67 | [Database Migrations with Flyway](67_Database%20Migrations%20with%20Flyway.md) |
| 68 | [Docker for Spring Boot](68_Docker%20for%20Spring%20Boot.md) |
| 69 | [Optimistic and Pessimistic Locking and Idempotency](69_Optimistic%20and%20Pessimistic%20Locking%20and%20Idempotency.md) |
| 70 | [Kafka Basics with Spring](70_Kafka%20Basics%20with%20Spring.md) |
| 71 | [Microservices with Spring Cloud and Resilience4j](71_Microservices%20with%20Spring%20Cloud%20and%20Resilience4j.md) |

> ⚠️ marks a **bug or gotcha found in the lecture code**. These make good "spot the bug" interview practice. ✅ "verified" means the behaviour was confirmed by actually running the lecture project.

## Practice
- Output-prediction questions on scopes, lazy/eager, and circular dependency: `../Notes_Of_Aditya/Lecture 06/PracticeQuestions.pdf` (answer key at the end).

## Top 10 rapid-fire interview questions (Lectures 1–10)
1. Spring vs Spring Boot? → Boot = auto-configured Spring (starters, embedded server, defaults).
2. IoC vs DI? → IoC is the principle; DI is the technique.
3. Which injection type is preferred and why? → Constructor: mandatory, `final`, testable, fails fast on cycles.
4. `@Component` vs `@Bean`? → Class-level auto-detection vs method-level manual creation (library classes).
5. Two beans of the same type? → `@Primary` (default) / `@Qualifier` (explicit; wins).
6. Singleton vs prototype? → One per bean definition vs new per request; prototype has no destroy callback.
7. Prototype inside a singleton? → Injected once; use `ObjectProvider` / `@Lookup` for fresh instances.
8. Bean lifecycle order? → Constructor → DI → Aware → `@PostConstruct` → ready → `@PreDestroy`.
9. What's inside `@SpringBootApplication`? → `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`.
10. `@Value` vs `@ConfigurationProperties`? → Single value vs type-safe group binding.

## Top 10 rapid-fire interview questions (Lectures 11–20)
1. Request flow in Spring MVC? → Tomcat → Filters → DispatcherServlet → HandlerMapping → HandlerAdapter (+ interceptors) → Controller → MessageConverter (Jackson) → response.
2. Servlet lifecycle? → constructor + `init()` once → `service()`/`doXxx()` per request (multi-threaded) → `destroy()` once.
3. JPA vs Hibernate vs Spring Data JPA? → Spec vs implementation vs repository abstraction on top.
4. How does `save()` choose INSERT or UPDATE? → New entity (null id) → persist; otherwise merge.
5. Why DTOs? → Decouple API from DB, prevent mass assignment, avoid leaking fields.
6. `@Valid` fails, which exception and status? → `MethodArgumentNotValidException` → 400.
7. Global exception handling? → `@RestControllerAdvice` + `@ExceptionHandler`; the most specific handler wins.
8. 400 vs 404 vs 409? → Bad input vs missing resource vs conflict with existing data.
9. Profile files vs `@Profile`? → Change values vs change which beans exist.
10. Filter vs Interceptor? → Servlet-level, before DispatcherServlet, can wrap req/res vs MVC-level, knows the handler method.

## Top 10 rapid-fire interview questions (Lectures 21–30)
1. Interceptor lifecycle? → `preHandle` (forward, `false` stops) → controller → `postHandle` (reverse, success only) → `afterCompletion` (reverse, always).
2. Aspect / advice / pointcut / join point? → Module / what runs / where / a method execution.
3. Five advice types? → `@Before`, `@AfterReturning`, `@AfterThrowing`, `@After` (finally), `@Around` (must return `proceed()`).
4. Why doesn't my `@Transactional` or aspect work on an internal call? → Self-invocation bypasses the proxy.
5. JDK vs CGLIB proxy? → Interface-based vs subclass; Boot defaults to CGLIB; `this()` vs `target()` differ with JDK proxies.
6. `Statement` vs `PreparedStatement`? → Parameterized, prevents SQL injection, handles types.
7. `query` vs `queryForObject`? → List (maybe empty) vs exactly one (throws `EmptyResultDataAccessException`).
8. Entity states? → Transient → managed → detached / removed; `merge` returns a new managed copy.
9. Flush vs commit? → Flush sends SQL inside the tx (can roll back); commit makes it durable.
10. Owning side / `mappedBy`? → The side with `@JoinColumn`/`@JoinTable` writes the FK; `mappedBy` names the owner's field.

## Top 10 rapid-fire interview questions (Lectures 31–40)
1. N+1 problem? → 1 parent query + N lazy-association queries; fix with `JOIN FETCH`, `@EntityGraph`, batch size, or DTOs (not EAGER).
2. `save()` internals? → `isNew` → `persist`, else `merge` (use the returned instance); managed entities update via dirty checking.
3. Page vs Slice? → Totals + count query vs `hasNext` only.
4. ACID? → Atomicity, Consistency, Isolation, Durability (bank transfer).
5. Default rollback rule? → RuntimeException/Error roll back; checked exceptions commit unless `rollbackFor`.
6. REQUIRED vs REQUIRES_NEW? → Join (shared fate) vs suspend + independent transaction; caught inner failure under REQUIRED → `UnexpectedRollbackException`.
7. Dirty / non-repeatable / phantom read? → Uncommitted data / same row changes / row set changes. MySQL defaults to RR, PostgreSQL to RC.
8. Spring Security login flow? → Filter → token → AuthenticationManager → DaoAuthenticationProvider → UserDetailsService + PasswordEncoder → SecurityContext.
9. JWT? → header.payload.signature; readable but tamper-evident; STATELESS; HS256 shared secret vs RS256 key pair.
10. OAuth vs OIDC; `@Mock` vs `@MockitoBean`? → Delegated authorization vs identity (ID token); plain Mockito vs mock bean in the Spring context.

## Top 10 rapid-fire interview questions (Spring gap topics 63–71)
1. `@Cacheable` vs `@CachePut` vs `@CacheEvict`? → Skip the method on a hit / always run and update the cache / remove entries. Proxy-based, so self-invocation bypasses them.
2. Cache stampede? → Many misses for a hot key hit the DB at once; fix with `sync = true`, locking, or early refresh + TTL jitter.
3. `@Async` pitfalls? → Needs `@EnableAsync`, a public method called through the proxy, a bounded custom executor; void-method exceptions need an `AsyncUncaughtExceptionHandler`.
4. `fixedRate` vs `fixedDelay`; multiple instances? → From the start vs from the end of the previous run; use ShedLock so a job runs once across pods.
5. Why always set HTTP client timeouts? → Hung calls exhaust request threads, causing cascading failure. Retry only idempotent, transient errors with backoff.
6. Liveness vs readiness? → Restart the pod vs remove it from the load balancer; keep external dependencies out of liveness.
7. Why Flyway over `ddl-auto=update`? → Versioned, reviewed, checksummed migrations; prod uses `validate`; expand/contract for zero downtime.
8. Optimistic vs pessimistic locking? → `@Version` check + retry/409 vs `SELECT … FOR UPDATE`; atomic conditional UPDATE + unique constraints for stock/seats.
9. Kafka ordering + delivery? → Ordered per partition (by key); at-least-once by default → idempotent consumers; outbox for DB + event consistency.
10. Circuit breaker states? → CLOSED → OPEN (fail fast + fallback) → HALF_OPEN trial calls; combine with timeouts and bulkheads; sagas for cross-service consistency.
