# Externalized Configuration: application.properties, @Value and @ConfigurationProperties

> **Source:** Lecture 10 (`Notes_Of_Aditya/Lecture 10/code/demo`)

## Why this topic matters
Hardcoded values (provider name, retry count, URLs, keys) force you to change code, recompile, rebuild, and redeploy. Config injection is used in every real project and comes up in interviews.

---

## 1. The problem
```java
@Component
public class PaymentService {
    private String providerName = "Razorpay";   // ❌ hardcoded
    private int retryCount = 3;
}
```
Changing Razorpay to Stripe means: edit code → recompile → rebuild → redeploy.

## 2. `application.properties` / `application.yml`
Default location: `src/main/resources/application.properties` (auto-loaded by Boot).
```properties
spring.application.name=demo

payment-property.type=paytm
payment-property.retry-count=5
payment-property.enabled=true
payment-property.timeout=3000
```
YAML equivalent:
```yaml
payment-property:
  type: paytm
  retry-count: 5
  enabled: true
  timeout: 3000
```

### "Externalized" configuration
The file inside `resources` is packaged into the JAR. The point is that Boot can also **override** values from outside the JAR without changing code:
- command-line args: `java -jar app.jar --payment-property.type=stripe`
- environment variables: `PAYMENTPROPERTY_TYPE=stripe`
- external `application.properties` next to the JAR, `config/` folder
- profile files: `application-dev.properties`, `application-prod.properties` (activated with `spring.profiles.active=prod`)

Simplified precedence (higher wins): **command-line > env variables > external file > packaged `application.properties`**.

## 3. `@Value`: one property at a time
```java
@Component
public class PaymentService {
    private final String providerName;
    private final int retryCount;

    public PaymentService(@Value("${payment.provider}") String providerName,
                          @Value("${payment.retry-count}") int retryCount) {
        this.providerName = providerName;
        this.retryCount = retryCount;
    }
}
```
`${key}` = look up `key` in the Spring **Environment** and inject it (type conversion happens automatically).

### Default values
```java
@Value("${paymentGateway.type:Razorpay}")     // use Razorpay if the key is missing
private String type;

@Value("${paymentGateway.retry-count:3}")
private int retryCount;
```
Without a default, a missing key fails at startup: `Could not resolve placeholder 'payment.provider'`.

## 4. `@ConfigurationProperties`: bind a group to an object
Code from the lecture:
```java
@Component
@ConfigurationProperties("payment-property")     // prefix
public class PaymentProperties {
    private String type;
    private int retryCount;       // ← payment-property.retry-count (relaxed binding)
    private boolean enabled;
    private int timeout;

    // getters + SETTERS are required for this JavaBean-style binding
    public String getType() { return type; }
    public void setType(String type) { this.type = type; }
    public int getRetryCount() { return retryCount; }
    public void setRetryCount(int retryCount) { this.retryCount = retryCount; }
    public boolean isEnabled() { return enabled; }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }
    public int getTimeout() { return timeout; }
    public void setTimeout(int timeout) { this.timeout = timeout; }
}
```
```java
@Component
public class PaymentGateway {
    private final PaymentProperties paymentProperties;

    public PaymentGateway(PaymentProperties paymentProperties) {   // inject the whole group
        this.paymentProperties = paymentProperties;
    }

    public void print() {
        System.out.println(paymentProperties.getType());
        System.out.println(paymentProperties.getRetryCount());
        System.out.println(paymentProperties.isEnabled());
        System.out.println(paymentProperties.getTimeout());
    }
}
```
Output:
```
paytm
5
true
3000
```

### Relaxed binding
`retry-count`, `retryCount`, `retry_count`, and `RETRY_COUNT` all bind to the field `retryCount`.
(Properties files use kebab-case; Java uses camelCase.)

### Other ways to register a properties class
```java
@EnableConfigurationProperties(PaymentProperties.class)   // on a @Configuration class
// or
@ConfigurationPropertiesScan                              // on the main class
// Immutable version (Boot 3): a record with constructor binding
@ConfigurationProperties("payment")
public record PaymentProps(String provider, int retryCount) { }
```
Add `@Validated` with `@NotBlank`, `@Min` etc. to validate config at startup.

## 5. `@Value` vs `@ConfigurationProperties`
| | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| Best for | 1–2 single values | A group of related values |
| Relaxed binding | Limited | ✅ Full |
| Type-safe POJO | ❌ | ✅ |
| SpEL (`#{...}`) | ✅ | ❌ |
| Validation (`@Validated`) | ❌ | ✅ |
| Readability at scale | Gets messy | Clean |

Rule: **1–2 values → `@Value`. A group → `@ConfigurationProperties`.**

---

## Common pitfalls
- Missing setters (non-record class) mean the fields stay null/0.
- Prefix typo means nothing binds and there's no error, so values silently default.
- `@Value` on a `static` field doesn't work.
- `@Value` in a class that is not a bean (created with `new`) is not injected.

## Interview Q&A
**Q1. What is externalized configuration?**
Keeping environment-specific or changeable values outside the code (properties/YAML, env vars, CLI args) so the same build runs in every environment.

**Q2. How do you read a property in Spring?**
`@Value("${key}")`, `@ConfigurationProperties(prefix)`, or `Environment.getProperty("key")`.

**Q3. How do you give a default value with `@Value`?**
`@Value("${key:default}")`.

**Q4. `@Value` vs `@ConfigurationProperties`?**
`@Value` is for single values and supports SpEL. `@ConfigurationProperties` gives type-safe binding of a prefix group with relaxed binding and validation.

**Q5. What is relaxed binding?**
Boot maps different naming styles (`retry-count`, `retry_count`, `RETRY_COUNT`) onto the same camelCase field.

**Q6. Which wins: `application.properties` or a command-line argument?**
The command-line argument. It has higher precedence than packaged properties.

**Q7. properties vs yml?**
Same purpose, different syntax. YAML is hierarchical and less repetitive for nested keys.

**Q8. How do you have different config per environment?**
Profiles: `application-{profile}.properties` plus `spring.profiles.active`.

## Quick revision
- Don't hardcode. Put values in `application.properties`.
- `@Value("${key:default}")` for single values.
- `@ConfigurationProperties("prefix")` + getters/setters for groups; relaxed binding maps kebab-case to camelCase.
- CLI args > env vars > packaged file.
