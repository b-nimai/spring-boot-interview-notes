# Generics

> **Part of:** Core Java (51–62)

## Why this topic matters
Generics appear everywhere (`List<T>`, `JpaRepository<T, ID>`, `ResponseEntity<T>`, `Optional<T>`). Interviewers ask about type erasure, wildcards, and the **PECS** rule.

---

## 1. Why generics
```java
List raw = new ArrayList();      // pre-Java 5
raw.add("a"); raw.add(1);
String s = (String) raw.get(1);  // ClassCastException at RUNTIME

List<String> list = new ArrayList<>();   // generic
list.add(1);                              // compile ERROR → type safety at compile time, no casts
```
Benefits: compile-time type safety, no casts, reusable algorithms.

## 2. Generic classes, interfaces, methods
```java
public class Box<T> {                         // generic class
    private T value;
    public void set(T v) { value = v; }
    public T get() { return value; }
}

public interface Repository<T, ID> {          // like Spring Data's JpaRepository<T, ID>
    Optional<T> findById(ID id);
    T save(T entity);
}

public static <T> T firstOrNull(List<T> list) {   // generic method: type param before the return type
    return list.isEmpty() ? null : list.get(0);
}
String s = firstOrNull(List.of("a", "b"));      // T inferred as String
```
Conventions: `T` type, `E` element, `K`/`V` key/value, `R` result, `ID`.

## 3. Bounded type parameters
```java
public static <T extends Comparable<T>> T max(List<T> list) {   // upper bound
    T best = list.get(0);
    for (T t : list) if (t.compareTo(best) > 0) best = t;
    return best;
}
<T extends Number & Comparable<T>>     // multiple bounds: class first, then interfaces
```

## 4. Invariance and wildcards
```java
List<Integer> ints = new ArrayList<>();
List<Number> nums = ints;          // ❌ compile error: generics are INVARIANT
// (else: nums.add(3.14) would corrupt a List<Integer>)
Object[] objs = new Integer[1];    // arrays are COVARIANT → objs[0] = "x" → ArrayStoreException at runtime
```
| Wildcard | Meaning | Read | Write |
|---|---|---|---|
| `List<?>` | Unknown type | as `Object` | only `null` |
| `List<? extends Number>` | Number or subtype (**producer**) | ✅ as `Number` | ❌ |
| `List<? super Integer>` | Integer or supertype (**consumer**) | only as `Object` | ✅ `Integer` |

### PECS: Producer Extends, Consumer Super
```java
double sum(List<? extends Number> src) {          // we READ from src (producer)
    double s = 0; for (Number n : src) s += n.doubleValue(); return s;
}
void fill(List<? super Integer> dst) {            // we WRITE into dst (consumer)
    dst.add(1); dst.add(2);
}
// JDK example:
public static <T> void copy(List<? super T> dest, List<? extends T> src)   // Collections.copy
sum(List.of(1, 2)); sum(List.of(1.5, 2.5));       // works for List<Integer> and List<Double>
fill(new ArrayList<Number>()); fill(new ArrayList<Object>());
```

## 5. Type erasure
Generics exist **only at compile time**. The compiler checks types, inserts casts, and erases type parameters to their bound (`Object` or the first bound).
```java
List<String> a = new ArrayList<>(); List<Integer> b = new ArrayList<>();
a.getClass() == b.getClass();      // true: both are just ArrayList at runtime
```
Consequences:
- ❌ `new T()`, `new T[10]`, `T.class`, `instanceof List<String>` (use `instanceof List<?>`).
- ❌ Overloading `m(List<String>)` and `m(List<Integer>)` (same erasure).
- ❌ Primitives as type args (`List<int>`), so boxing is needed (`List<Integer>`; streams have `IntStream` to avoid it).
- ❌ Static fields of type `T`.
- Workaround for the runtime type: pass `Class<T> clazz` (e.g. `objectMapper.readValue(json, User.class)`) or a **super type token** (`new TypeReference<List<User>>() {}` in Jackson, `ParameterizedTypeReference` in Spring `RestTemplate`).
- The compiler generates **bridge methods** to keep polymorphism working after erasure.
- Heap pollution / `@SafeVarargs` on generic varargs.

> 🟦 TS generics are also erased (to JS), but TS doesn't insert runtime casts. Java does, so a bad unchecked cast fails later with `ClassCastException` where the value is used.

## 6. Raw types and unchecked warnings
`List list = new ArrayList();` is a raw type that bypasses checks (only for legacy interop). Treat "unchecked" warnings seriously; use `@SuppressWarnings("unchecked")` only in narrow, proven-safe spots.

---

## Interview Q&A
**Q1. What is type erasure?**
Generic type info is removed at compile time (replaced by bounds plus casts), so `List<String>` and `List<Integer>` are the same class at runtime.

**Q2. Why can't you create `new T()` or `new T[]`?**
`T` is unknown at runtime due to erasure. Pass a `Class<T>` or a `Supplier<T>` instead.

**Q3. Is `List<Integer>` a subtype of `List<Number>`?**
No, generics are invariant. Use `List<? extends Number>`.

**Q4. Explain PECS.**
Producer Extends (read from `? extends T`), Consumer Super (write to `? super T`).

**Q5. `List<?>` vs `List<Object>`?**
`List<?>` is a list of some unknown type (you can't add except null) and accepts any `List<X>`. `List<Object>` accepts only `List<Object>`, but you can add anything.

**Q6. Arrays vs generics regarding type safety?**
Arrays are covariant and reified (runtime `ArrayStoreException`). Generics are invariant and erased (compile-time errors).

**Q7. How does Jackson deserialize `List<User>` if types are erased?**
Through a `TypeReference` subclass that captures the generic type in the class's superclass signature (readable via reflection).

## Quick revision
- Compile-time safety, no casts; `<T>` classes and methods; bounds with `extends` (& multiple).
- Invariant generics vs covariant arrays.
- `?`, `? extends` (read/producer), `? super` (write/consumer): **PECS**.
- Erasure: no `new T()`, no `T.class`, no `instanceof List<String>`, no primitives; use `Class<T>`/TypeReference.
