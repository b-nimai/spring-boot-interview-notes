# OOP in Java: Classes, Interfaces, Inheritance and Polymorphism

> **Part of:** Core Java (51–62) · Foundation for LLD.

## Why this topic matters
Every LLD round and most Java interviews test the four pillars, overloading vs overriding, abstract class vs interface, and composition vs inheritance. Spring itself relies on interfaces + polymorphism (DI, proxies).

---

## 1. The four pillars
| Pillar | Meaning | Java mechanism |
|---|---|---|
| **Encapsulation** | Hide state, expose behaviour | `private` fields + methods that enforce invariants |
| **Abstraction** | Expose *what*, hide *how* | Interfaces, abstract classes |
| **Inheritance** | Reuse/extend a type | `extends` (single class), `implements` (many interfaces) |
| **Polymorphism** | One interface, many forms | Overriding (runtime), overloading (compile-time) |

### Encapsulation done right
```java
public class BankAccount {
    private BigDecimal balance = BigDecimal.ZERO;          // hidden state
    public void withdraw(BigDecimal amt) {                 // behaviour enforces the rule
        if (amt.compareTo(balance) > 0) throw new IllegalStateException("Insufficient funds");
        balance = balance.subtract(amt);
    }
    public BigDecimal getBalance() { return balance; }     // no setBalance: invariant protected
}
```
Getters/setters for every field isn't encapsulation. Expose **behaviour**.

## 2. Classes, objects, constructors
```java
public class User {
    private final String name;
    private int age;
    public User(String name) { this(name, 0); }                 // constructor chaining with this(...)
    public User(String name, int age) { this.name = name; this.age = age; }
}
```
- No constructor written means the compiler adds a default no-arg one. Write any constructor and it doesn't.
- `this(...)` / `super(...)` must be the **first statement**. If omitted, `super()` is inserted, so the parent needs a no-arg constructor.
- Initialization order: static blocks (once) → instance initializers/field initializers → constructor body. For subclasses, the parent's constructor runs first.

## 3. Inheritance
```java
class Animal {
    protected String name;
    Animal(String name) { this.name = name; }
    void speak() { System.out.println("..."); }
}
class Dog extends Animal {
    Dog(String name) { super(name); }
    @Override void speak() { System.out.println(name + ": Woof"); }
}
```
- **Single class inheritance** (avoids the diamond problem for state); multiple interfaces allowed.
- Everything extends `Object` (`equals`, `hashCode`, `toString`, `getClass`, `wait/notify`, `clone`).
- `final` classes and methods can't be extended or overridden; `private` methods aren't inherited (not overridable); `static` methods are **hidden**, not overridden.

## 4. Polymorphism
### Runtime (overriding / dynamic dispatch)
```java
Animal a = new Dog("Rex");   // reference type Animal, object type Dog
a.speak();                   // "Rex: Woof" → method chosen by the OBJECT type at runtime
```
### Compile-time (overloading)
```java
int add(int a, int b)            {...}
double add(double a, double b)   {...}
int add(int a, int b, int c)     {...}   // same name, different parameter lists
```
| Overloading | Overriding |
|---|---|
| Same class (or subclass), **different parameters** | Subclass, **same signature** |
| Resolved at **compile time** (static binding) | Resolved at **runtime** (dynamic dispatch) |
| Return type can differ (if params differ) | Same or **covariant** (subtype) return |
| Any access | Can't reduce visibility; can't throw broader checked exceptions |
| `static` methods can be overloaded | `static` methods are hidden, not overridden |
Always annotate `@Override` (the compiler catches signature typos).
**Fields aren't polymorphic**: `a.name` uses the reference type's field.

### Casting
```java
if (a instanceof Dog d) d.fetch();      // pattern matching (Java 16+): safe downcast
Dog d2 = (Dog) a;                       // ClassCastException if a isn't a Dog
```

## 5. Abstract classes vs interfaces
```java
public interface PaymentGateway {
    PaymentResult pay(BigDecimal amount);                 // abstract (public)
    default boolean supportsRefund() { return false; }    // default method (Java 8)
    static PaymentGateway noop() { return a -> PaymentResult.ok(); }   // static (8)
    // private helper methods allowed (Java 9)
}

public abstract class AbstractGateway implements PaymentGateway {
    protected final HttpClient http;                      // state + constructor
    protected AbstractGateway(HttpClient http) { this.http = http; }
    public final PaymentResult pay(BigDecimal amount) {   // template method
        validate(amount); return doPay(amount);
    }
    protected abstract PaymentResult doPay(BigDecimal amount);
}
```
| | Abstract class | Interface |
|---|---|---|
| Inheritance | Single (`extends`) | Multiple (`implements`) |
| State | Instance fields, constructors | Only `public static final` constants |
| Methods | Any access; abstract + concrete | Abstract, `default`, `static`, `private` |
| Purpose | **"is-a"** with shared code/state (template) | **Capability/contract** ("can-do") |
Default-method conflict (two interfaces with the same default): the class **must override** it and can call `A.super.m()`.
**Functional interface** = exactly one abstract method, so it can be a lambda target ([58](58_Java%208%2B%20-%20Lambdas%2C%20Functional%20Interfaces%2C%20Streams%20and%20Optional.md)).

> 🟦 TS interfaces are compile-time only (structural typing). Java interfaces are **nominal** runtime types: a class must explicitly `implements` them.

## 6. Composition over inheritance
```java
// ❌ inheritance for reuse → fragile coupling (Stack extends Vector is a famous mistake)
class InstrumentedSet<E> extends HashSet<E> { ... }   // breaks when HashSet internals change

// ✅ composition + delegation
class InstrumentedSet<E> implements Set<E> {
    private final Set<E> inner;   // HAS-A
    ...
}
```
Prefer **HAS-A** (composition) unless there's a true **IS-A** relationship that respects Liskov. Spring DI *is* composition.

## 7. SOLID (LLD preview)
| Principle | One line |
|---|---|
| **S**ingle Responsibility | One reason to change |
| **O**pen/Closed | Extend via new classes, don't modify old ones (strategy pattern, interfaces) |
| **L**iskov Substitution | A subtype must work anywhere the parent works (Square-extends-Rectangle violates it) |
| **I**nterface Segregation | Small, focused interfaces |
| **D**ependency Inversion | Depend on abstractions; inject implementations (Spring DI) |

## 8. Other class kinds
- **Nested static class**: no outer instance (builders, `Map.Entry`).
- **Inner class**: holds a reference to the outer instance (can leak memory).
- **Anonymous class**: an inline implementation (mostly replaced by lambdas).
- **Enum**: a type-safe constant set. Can have fields, constructors, methods, and abstract methods per constant. A singleton via `enum` is serialization- and reflection-safe.
- **Record** (16) and **sealed** classes (17): see [53](53_equals%2C%20hashCode%2C%20Immutability%20and%20Records.md), [62](62_Modern%20Java%209-21%20Features.md).

---

## Interview Q&A
**Q1. Overloading vs overriding?**
Different parameters, resolved at compile time vs same signature in a subclass, resolved at runtime.

**Q2. Can we override static or private methods?**
No. Static methods are hidden (resolved by reference type); private methods aren't inherited.

**Q3. Abstract class vs interface: when to use which?**
Interface for contracts and multiple capabilities; abstract class to share state/code among closely related classes (template method).

**Q4. Why doesn't Java support multiple class inheritance?**
The diamond problem with state and constructors. Interfaces give multiple type inheritance, and default-method conflicts must be resolved explicitly.

**Q5. What is dynamic method dispatch?**
The JVM selects the overridden method by the object's runtime type (virtual call through the vtable).

**Q6. Composition vs inheritance?**
Composition (has-a, delegation) gives loose coupling and runtime flexibility. Inheritance (is-a) couples tightly to parent internals. Prefer composition.

**Q7. Can a constructor be private? Why?**
Yes: for singletons, static factories, utility classes, and builders.

**Q8. What's the Liskov Substitution Principle? Example of a violation?**
Subtypes must honour the parent's contract. `Square extends Rectangle`, where `setWidth` also changes the height, breaks callers.

## Quick revision
- Encapsulation (protect invariants), abstraction (interfaces), inheritance (single class, multiple interfaces), polymorphism (runtime override, compile-time overload).
- `@Override` always; static = hidden; fields aren't polymorphic.
- Interface = contract (default/static/private methods); abstract class = shared state + template.
- Prefer composition; SOLID is the bridge to LLD.
