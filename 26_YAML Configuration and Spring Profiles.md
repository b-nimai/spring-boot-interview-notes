# YAML Configuration and Spring Profiles

> **Source:** Lecture 18 (`Notes_Of_Aditya/Lecture 18/code/profileDemo`)
> **Related:** [16_Externalized Configuration](16_Externalized%20Configuration%20-%20application.properties%2C%20Value%20and%20ConfigurationProperties.md)

## Why this topic matters
The same JAR has to run in dev, test, staging, and prod with different DBs, ports, keys, and even different beans. Profiles are how Spring does this. You'll be asked how to activate them and the difference between profile files and `@Profile`.

---

## 1. Application = Code + Configuration
Hardcoding (`String password = "root"`) means you rebuild for every change, can't run the same code in several environments, and leak secrets to Git.
Goal: **same code, same build, different configuration**.

## 2. `.properties` vs `.yml`
```properties
app.name=Student Management App
app.support.email=support@studentapp.com
app.support.phone=9999999999
```
```yaml
app:
  name: Student Management App
  support:
    email: support@studentapp.com
    phone: 9999999999
```
YAML is **the same keys** written hierarchically. Spring flattens it to `app.support.email` internally, so `@Value("${app.support.email}")` works for both.

### YAML rules
1. `key: value`, with a **space after the colon**.
2. **Indentation defines nesting.** Use **spaces only, never tabs** (2 spaces is the convention).
3. Same-level keys get the same indentation.
4. `#` starts a comment.
5. Quote values with special characters (`"${...}"`, `:`, `#`, leading `*`).
6. Lists:
```yaml
app:
  supported-cities:
    - Delhi
    - Mumbai
  team-members:            # list of objects → best bound with @ConfigurationProperties
    - name: Rohit
      role: Backend Mentor
    - name: Aditya
      role: Spring Mentor
```
> Pick **one** format per project. If both `application.properties` and `application.yml` exist, both are loaded, and for duplicate keys `.properties` wins, which is confusing.

### Reading lists with `@Value`
```java
@Value("${app.welcome.users}")          // app.welcome.users=Aditya,Rohit,Rohan,Sohan
private List<String> names;             // Spring's conversion splits on commas → [Aditya, Rohit, Rohan, Sohan]

@Value("${app.supported-cities[0]}")    // YAML list item by index
private String firstCity;

@Value("#{'${app.supported-cities}'.split(',')}")   // SpEL alternative
private List<String> cities;
```

## 3. Profiles
A **profile** is a named environment: `dev`, `test`, `staging`, `prod`.

### Profile-specific files
```
src/main/resources
├── application.properties          ← common/defaults (always loaded)
├── application-dev.properties
├── application-staging.properties
└── application-prod.properties
```
Lecture files:
```properties
# application.properties
spring.application.name=profileDemo
app.welcome.message=LOCAL: Hello, welcome to my application
app.welcome.code=200
app.welcome.users=Aditya,Rohit,Rohan,Sohan
#spring.profiles.active=dev

# application-dev.properties
server.port=9090
app.welcome.message=DEV: Hello, welcome to my application

# application-prod.properties
server.port=8191
app.welcome.message=PROD: Hello, welcome to my application
```
**Merge rule:** active profile = `dev` loads `application.properties` **+** `application-dev.properties`. **The profile file overrides duplicate keys.** Keys not in the profile file (`app.welcome.code`) come from the common file.

```java
@RestController
@RequestMapping("api/demo")
public class DemoController {
    @Value("${app.welcome.message}") private String message;
    @Value("${app.welcome.code}")    private Integer code;
    @Value("${app.welcome.users}")   private List<String> names;

    @GetMapping("/greet")
    public ResponseEntity<String> greet() { return ResponseEntity.ok(message + " : " + code); }
}
```
| Active profile | Port | `GET /api/demo/greet` |
|---|---|---|
| none (`default`) | 8080 | `LOCAL: Hello, welcome to my application : 200` |
| `dev` | 9090 | `DEV: Hello, welcome to my application : 200` |
| `staging` | 8080 | `STAGING: Hello, … : 200` |
| `prod` | 8191 | `PROD: Hello, … : 200` |

### Activating a profile
```properties
spring.profiles.active=dev                                   # in application.properties (demo only)
```
```bash
java -jar app.jar --spring.profiles.active=prod               # command line ✅
SPRING_PROFILES_ACTIVE=prod java -jar app.jar                 # env var ✅ (Docker/K8s/CI)
java -Dspring.profiles.active=prod -jar app.jar               # JVM system property
mvn spring-boot:run -Dspring-boot.run.profiles=dev            # Maven during development
```
IntelliJ: Run Configuration → *Active profiles*. Multiple profiles: `--spring.profiles.active=dev,swagger`.
> **Don't hardcode `spring.profiles.active=prod` inside the JAR.** Let the environment decide, so one artifact can be promoted dev → staging → prod.

Single-file alternative (YAML multi-document):
```yaml
spring:
  application:
    name: demo
---
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 9090
```

## 4. `@Profile`: different beans per environment
```java
public interface NotificationService { String send(); }

@Service
@Profile({"dev", "default", "staging"})       // "default" = when NO profile is active
public class DummyNotificationServiceImpl implements NotificationService {
    public String send() { return "Here is a dummy notification"; }   // no real SMS/email
}

@Service
@Profile("prod")
public class NotificationServiceImpl implements NotificationService {
    public String send() { return "Here is a notification"; }         // real provider
}

@RestController
@RequestMapping("api/notification")
public class NotificationController {
    private final NotificationService notificationService;            // exactly ONE bean exists per profile
    public NotificationController(NotificationService ns) { this.notificationService = ns; }

    @PostMapping
    public ResponseEntity<String> sendNotification() { return ResponseEntity.ok(notificationService.send()); }
}
```
| Profile | Bean created | `POST /api/notification` |
|---|---|---|
| none | Dummy | `Here is a dummy notification` |
| `dev` / `staging` | Dummy | `Here is a dummy notification` |
| `prod` | Real | `Here is a notification` |

Without `@Profile`, both beans exist, so you get `NoUniqueBeanDefinitionException`.
If an unlisted profile is active (e.g. `test`), **no** bean exists, so you get `NoSuchBeanDefinitionException`. Every profile must be covered.

`@Profile` expressions: `@Profile("!prod")` (not prod), `@Profile("dev | test")`, `@Profile("prod & eu")`. It also works on `@Configuration` and `@Bean` methods.

## 5. Profile files vs `@Profile`
| Need | Use |
|---|---|
| Same class, **different values** (port, DB URL, log level, messages, feature flags) | `application-{profile}.properties/yml` |
| **Different implementation/bean** (fake vs real email, local vs S3 storage, mock vs real payment) | `@Profile` |

## 6. Common mistakes
- Hardcoding environment values in Java.
- Keeping both `.properties` and `.yml` for the same keys.
- Baking `spring.profiles.active=prod` into the JAR.
- Using many `@Value`s for nested config. Use `@ConfigurationProperties` instead.
- Committing prod secrets in `application-prod.properties`. Use env vars: `spring.datasource.password=${DB_PASSWORD}`.
- Typos in keys: the lecture's `Spring.application.name` has a capital **S**. Use lowercase kebab-case keys (`spring.application.name`) as the documented canonical form.

---

## Interview Q&A
**Q1. What are Spring profiles?**
Named environment configurations that let the same application load different properties and beans (`dev`, `test`, `prod`).

**Q2. How do you activate a profile?**
`spring.profiles.active` via a CLI arg, env var `SPRING_PROFILES_ACTIVE`, JVM property, properties file, IDE config, or `@ActiveProfiles` in tests.

**Q3. If a key exists in both `application.properties` and `application-dev.properties`, which wins when `dev` is active?**
The profile-specific file.

**Q4. What is the `default` profile?**
The profile that's active when no profile is explicitly set. `@Profile("default")` beans load only then.

**Q5. Profile-specific files vs `@Profile`?**
Files change **values**. `@Profile` changes **which beans exist**.

**Q6. properties vs YAML?**
Same keys and semantics. YAML is hierarchical and better for nested or list config, but indentation-sensitive. `.properties` wins on duplicate keys.

**Q7. How do you keep secrets out of config files?**
Use placeholders resolved from env vars or a vault (`${DB_PASSWORD}`), Spring Cloud Config, Kubernetes Secrets, etc.

**Q8. How do you activate a profile in tests?**
`@ActiveProfiles("test")` on the test class.

## Quick revision
- YAML = nested `.properties`; spaces only; `- item` for lists.
- `application.properties` (common) + `application-{p}.properties` (override).
- Activate from outside: `--spring.profiles.active=prod` / `SPRING_PROFILES_ACTIVE`.
- `@Profile` picks beans; `"default"` covers "no profile"; cover every profile.
