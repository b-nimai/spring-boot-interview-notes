# Tight vs Loose Coupling

> **Source:** Lecture 04 (`Notes_Of_Aditya/Lecture 04/code/CoreDemo`)

## Why this topic matters
Loose coupling is the **reason Spring Core exists**. Before you learn IoC/DI, you need to see the design problem they solve.

---

## 1. What is a dependency?
A **dependency** is something a class needs to do its work.
`OrderService` needs something to send a notification, so the notifier is its dependency.

## 2. Tight coupling
```java
class OrderService {
    private EmailService emailService = new EmailService();   // creates its own concrete dependency

    public void placeOrder() {
        System.out.println("Order placed");
        emailService.sendEmail();
    }
}
```
The requirement changes: send SMS instead of email. Now you **must edit `OrderService`**:
```java
class OrderService {
    private SmsService smsService = new SmsService();          // modified business class
    ...
}
```
> **Tight coupling** = a class depends directly on a **specific concrete class**, so a change in the dependency forces a change in the class.

Real-life analogy: "I can travel only in *this* car with *this* driver" (tight) vs "I need *transportation*: car, bus, train or cab" (loose).

## 3. Step 1: Program to an interface
```java
public interface NotificationService {
    void sendNotification();
}

public class EmailService implements NotificationService {
    @Override public void sendNotification() { System.out.println("Email notification sent"); }
}
public class SmsService implements NotificationService {
    @Override public void sendNotification() { System.out.println("SMS notification sent"); }
}
public class PopUpNotificationService implements NotificationService {
    @Override public void sendNotification() { System.out.println("Pop-up notification sent"); }
}
```
```java
class OrderService {
    private NotificationService notification = new EmailService(); // still "new EmailService()"
}
```
Better, but **the interface alone is not enough**. `OrderService` still **decides which implementation to create**.

> Creating objects is not the problem. **Creating them in the wrong place** is the problem.

At this point `OrderService` has two jobs: order logic **and** choosing or creating the notifier. That violates:
- **S**ingle Responsibility Principle
- **O**pen/Closed Principle (you have to modify the class to extend its behaviour)

## 4. Step 2: Receive the dependency from outside (loose coupling)
```java
public class OrderService {
    NotificationService notification;

    public OrderService(NotificationService notification) {   // constructor injection
        this.notification = notification;
    }
    public OrderService() { }

    public void setNotification(NotificationService notification) { // setter injection
        this.notification = notification;
    }

    public void placeOrder() {
        System.out.println("Order placed");
        notification.sendNotification();
    }
}
```
```java
public class Main {
    public static void main(String[] args) {
        NotificationService notification = new EmailService(); // decision made OUTSIDE
        OrderService order = new OrderService();
        order.setNotification(notification);
        order.placeOrder();
    }
}
// A class should ask for what it needs and not build everything itself.
```
Output:
```
Order placed
Email notification sent
```
Switching to SMS now means changing only `Main`. **`OrderService` stays untouched.**

## 5. Benefits of loose coupling
| Benefit | How |
|---|---|
| Easy to change | Pass a different implementation |
| Easy to test | Pass a `FakeEmailService` that only prints "Dummy Email sent" |
| Reusable | The same `OrderService` works with Email, SMS, Pop-up, WhatsApp… |

```java
public class FakeEmailService implements NotificationService {
    @Override public void sendNotification() { System.out.println("Dummy Email sent"); }
}
```

---

## Interview Q&A
**Q1. What is tight coupling? Give an example.**
A class directly creates or depends on a concrete class, e.g. `new EmailService()` inside `OrderService`. Changing to SMS forces you to edit `OrderService`.

**Q2. Is using an interface enough for loose coupling?**
No. If the class still does `new ConcreteImpl()`, it's still choosing the implementation. You also need to move object creation outside (DI).

**Q3. Which SOLID principles does tight coupling violate?**
SRP (the class does business logic *and* builds its dependencies) and OCP (the class must be modified to change behaviour). Programming to an interface also relates to DIP (Dependency Inversion Principle).

**Q4. How does loose coupling help testing?**
You can inject fake or mock implementations, so no real email or SMS is sent.

## Quick revision
- Dependency is normal. A **tight** dependency is the problem.
- Interface + injecting from outside = loose coupling.
- "Don't create your dependency, receive it."
