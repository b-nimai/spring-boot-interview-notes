# Resolving Multiple Beans: @Primary and @Qualifier

> **Source:** Lecture 05 (`Notes_Of_Aditya/Lecture 05/code/SpringCoreDemo` → `payment/`, `AppConfig.java`)

## Why this topic matters
When an interface has multiple implementations (UPI, Card…), Spring can't guess which one to inject. You must know how to tell it, and which hint wins.

---

## 1. The ambiguity problem
```java
public interface PaymentService { void pay(); }

@Component public class UpiPayment  implements PaymentService { public void pay() { System.out.println("Paying by UPI"); } }
@Component public class CardPayment implements PaymentService { public void pay() { System.out.println("Paying via card"); } }

@Component
public class OrderService {
    public OrderService(PaymentService paymentService) { ... }   // which one?
}
```
Result: `NoUniqueBeanDefinitionException: expected single matching bean but found 2: upiPayment, cardPayment`

## 2. `@Primary`: the default choice
```java
@Primary
@Component
public class UpiPayment implements PaymentService { ... }
```
"If many candidates exist, prefer this one by default."

## 3. `@Qualifier`: the explicit choice
```java
@Component
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(@Qualifier("cardPayment") PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```
**Default bean names:** the class name with its first letter lower-cased (`CardPayment` → `cardPayment`).
⚠️ Acronyms behave differently: `UPIPaymentService` stays `UPIPaymentService`, because if the first two letters are uppercase the name is kept as-is. To avoid surprises, **give explicit names**:
```java
@Component("upi")  public class UpiPayment  implements PaymentService { }
@Component("card") public class CardPayment implements PaymentService { }

public OrderService(@Qualifier("card") PaymentService paymentService) { ... }
```

### `@Qualifier` works with every injection style
```java
// Field
@Autowired @Qualifier("upi")
private PaymentService paymentService;

// Setter
@Autowired
public void setPaymentService(@Qualifier("card") PaymentService p) { this.paymentService = p; }
```

## 4. With `@Bean` methods (code from the lecture)
```java
@Configuration
@ComponentScan("in.coderarmy")
public class AppConfig {

    @Bean
    @Qualifier("cp")                       // qualifier label on the bean
    public PaymentService createCardPayment() { return new CardPayment(); }

    @Bean
    @Qualifier("upi")
    public PaymentService createUpiPayment() { return new UpiPayment(); }

    @Bean
    public OrderService createOrderService(@Qualifier("cp") PaymentService paymentService) {
        return new OrderService(paymentService);   // Card payment is injected
    }
}
```
Output: `Paying via card` then `Order placed`.
(`@Bean @Primary` works too.)

## 5. Priority rule
> **`@Qualifier` (specific) beats `@Primary` (default).**

Spring's resolution order when there are several candidates:
1. `@Qualifier` match
2. `@Primary` bean
3. Parameter or field **name** matching a bean name (e.g. a parameter named `cardPayment`)
4. Otherwise → `NoUniqueBeanDefinitionException`

## 6. Inject all implementations
```java
public CheckoutService(List<PaymentService> all) { ... }              // every PaymentService bean
public CheckoutService(Map<String, PaymentService> byName) { ... }    // beanName → bean (strategy pattern)
```

---

## Interview Q&A
**Q1. What happens if two beans of the same type exist and you autowire by type?**
Startup fails with `NoUniqueBeanDefinitionException`, unless it's resolved by `@Primary`, `@Qualifier`, or a matching parameter name.

**Q2. `@Primary` vs `@Qualifier`?**
`@Primary` sets a global default for a type. `@Qualifier` picks a specific bean at a specific injection point. If both apply, `@Qualifier` wins.

**Q3. What is the default bean name for `@Component` and for `@Bean`?**
`@Component`: the class name in camelCase (acronyms like `UPIService` are kept as-is). `@Bean`: the method name.

**Q4. How would you inject all implementations of an interface?**
Inject a `List<Interface>` or `Map<String, Interface>`.

**Q5. Can `@Primary` be used on more than one bean of the same type?**
No. Two primaries for the same type is still ambiguous and throws an exception.

## Quick revision
- Many implementations → ambiguity.
- `@Primary` = default, `@Qualifier("name")` = exact choice. Qualifier wins.
- Give explicit names with `@Component("name")` / `@Bean("name")`.
