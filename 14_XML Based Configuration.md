# XML Based Configuration

> **Source:** Lecture 08 (`Notes_Of_Aditya/Lecture 08/code/XMLBasedConfigDemo`)

## Why this topic matters
Legacy enterprise projects still have `applicationContext.xml` files. XML also makes the wiring **explicit**, which helps you understand what annotations do for you.

---

## 1. Configuration metadata
XML, annotations, and Java config are just **three ways to give instructions**. Internally all of them become **BeanDefinition** objects.
```
XML / Annotations / Java Config  →  BeanDefinitions  →  IoC Container (create + wire + manage)
```

## 2. First XML bean
Put the file in `src/main/resources` (it ends up on the classpath).
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="upiPaymentService" class="in.strikes.payment.UPIPaymentService"/>
</beans>
```
- `id` = bean name; `class` = fully qualified class name.
- Equivalent to `@Component("upiPaymentService")`.

```java
ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
```
Beans are **singleton + eager**, so constructors run as soon as the context loads.

## 3. Getting beans
```java
OrderService o1 = (OrderService) context.getBean("orderService");        // by name (cast needed)
OrderService o2 = context.getBean("orderService", OrderService.class);   // by name + type ✅
OrderService o3 = context.getBean(OrderService.class);                   // by type (needs exactly 1 match)
```
If there are two beans of the same type, getting by type throws `NoUniqueBeanDefinitionException`.

### Names
```xml
<bean id="paymentService" name="payService, gatewayService" class="..."/>  <!-- aliases -->
<alias name="paymentService" alias="txService"/>                          <!-- external alias -->
```
- With no `id`, you can only get the bean by type (Spring generates a name).
- Duplicate `id`s in one file are invalid.

## 4. Dependency injection in XML
XML injects only through **constructors** or **setters**. There's no field injection.

### Constructor injection
```xml
<bean id="upiPaymentService" class="in.strikes.payment.UPIPaymentService"/>

<bean id="orderService" class="in.strikes.OrderService">
    <constructor-arg ref="upiPaymentService"/>          <!-- ref = another bean -->
</bean>
```
Simple values and argument matching:
```xml
<bean id="paymentService" class="in.strikes.PaymentService">
    <constructor-arg index="0" value="Razorpay"/>       <!-- by index -->
    <constructor-arg index="1" value="3"/>
    <!-- or name="providerName" / type="java.lang.String" -->
</bean>
```

### Setter injection
```xml
<bean id="orderService" class="in.strikes.OrderService">
    <property name="paymentService" ref="paymentService"/>   <!-- calls setPaymentService(...) -->
</bean>
```
`name` = property name (Spring looks for `set` + Name). From the lecture's commented code: `<property name="PaymentServiceBean" .../>` maps to `setPaymentServiceBean(...)`.

| Attribute | Injects |
|---|---|
| `value` | Simple value (String, int, boolean) |
| `ref` | Another bean |

### Collections
```xml
<bean id="userService" class="in.strikes.UserService">
    <constructor-arg name="usernames">
        <list>
            <value>Aditya</value>
            <value>Rohit</value>
        </list>
    </constructor-arg>
</bean>

<constructor-arg name="userBalances">
    <map>
        <entry key="Aditya" value="2.0"/>
        <entry key="Rohit"  value="2.5"/>
    </map>
</constructor-arg>
```

## 5. Multiple beans of the same type
```xml
<!-- 1. Explicit ref (clearest) -->
<constructor-arg ref="razorpayPaymentService"/>
<!-- 2. primary -->
<bean id="razorpayPaymentService" class="..." primary="true"/>
<!-- 3. exclude from autowiring -->
<bean id="stripePaymentService" class="..." autowire-candidate="false"/>
```

## 6. XML autowiring
```xml
<bean id="orderService" class="in.strikes.OrderService" autowire="byName"/>
```
| Mode | Matches by |
|---|---|
| `byName` | Setter property name == bean id |
| `byType` | Property type (needs exactly one match) |
| `constructor` | Constructor parameter types |
| `no` (default) | Nothing; wire explicitly |

## 7. Scope and lifecycle in XML (lecture's `beans2.xml`)
```xml
<bean id="userService"
      class="in.strikes.UserService"
      init-method="init"
      destroy-method="cleanup"
      scope="prototype"/>
```
```java
public class UserService {
    public UserService() { System.out.println("UserService created"); }
    public void init()    { System.out.println("Post construct phase"); }
    public void cleanup() { System.out.println("Pre Destroy phase"); }
}
```
```java
ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("appConfig.xml");
UserService user = context.getBean(UserService.class);
context.close();
```
Output:
```
UserService created
Post construct phase
                      ← no "Pre Destroy phase" because scope is prototype
```
Remove `scope="prototype"` (making it a singleton) and `Pre Destroy phase` is printed on `close()`.

## 8. Splitting XML files with `<import>`
```xml
<!-- appConfig.xml -->
<beans ...>
    <import resource="beans.xml"/>
    <import resource="beans2.xml"/>
</beans>
```
```java
new ClassPathXmlApplicationContext("appConfig.xml");   // load only the main file
```

## 9. Mixing XML with annotations
```xml
<beans xmlns:context="http://www.springframework.org/schema/context" ...>
    <context:component-scan base-package="in.strikes"/>
</beans>
```
Or from Java config: `@ImportResource("classpath:beans.xml")`.

---

## Interview Q&A
**Q1. How do you load an XML config?**
`new ClassPathXmlApplicationContext("beans.xml")` (from the classpath) or `FileSystemXmlApplicationContext` (from a file path).

**Q2. `<constructor-arg>` vs `<property>`?**
`<constructor-arg>` is constructor injection. `<property>` is setter injection and needs a matching `setXxx` method.

**Q3. `value` vs `ref`?**
`value` is a literal. `ref` points to another bean.

**Q4. XML autowiring modes?**
`no`, `byName`, `byType`, `constructor`.

**Q5. How do you define init/destroy callbacks in XML?**
`init-method` and `destroy-method` attributes on `<bean>` (or `default-init-method` on `<beans>`).

**Q6. Can XML and annotations be used together?**
Yes. Use `<context:component-scan>` in XML or `@ImportResource` in Java config. Everything ends up as BeanDefinitions.

**Q7. Why learn XML today?**
Legacy codebases, and it makes Spring's wiring explicit and easy to reason about.

## Quick revision
- `<bean id class>` registers a bean. `ClassPathXmlApplicationContext` loads the file.
- Constructor → `<constructor-arg>`, setter → `<property>`. `value` vs `ref`.
- `scope`, `init-method`, `destroy-method`, `primary`, `autowire` are `<bean>` attributes.
- `<import>` splits files. Prototype → no destroy call.
