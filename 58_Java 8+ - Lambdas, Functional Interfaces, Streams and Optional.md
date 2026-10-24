# Java 8+: Lambdas, Functional Interfaces, Streams and Optional

> **Part of:** Core Java (51–62)

## Why this topic matters
Streams are used everywhere in modern Spring code (`list.stream().map(this::toDto).toList()`), and "write this with streams" is a common live-coding prompt. You'll also be asked `map` vs `flatMap`, lazy evaluation, and Optional misuse.

---

## 1. Lambdas
```java
Comparator<String> byLen = (a, b) -> Integer.compare(a.length(), b.length());
Runnable r = () -> System.out.println("hi");
Function<Integer, Integer> square = x -> x * x;
```
- An implementation of a **functional interface** (exactly one abstract method).
- Captured local variables must be **effectively final** (no reassignment after capture).
- `this` inside a lambda = the enclosing instance (unlike anonymous classes).
> 🟦 Same feel as TS arrow functions, but Java lambdas must target a functional-interface type, and captured locals can't be reassigned.

### Method references
| Kind | Example | Lambda equivalent |
|---|---|---|
| Static | `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| Bound instance | `System.out::println` | `x -> System.out.println(x)` |
| Unbound instance | `String::toUpperCase` | `s -> s.toUpperCase()` |
| Constructor | `ArrayList::new` | `() -> new ArrayList<>()` |

## 2. Core functional interfaces (`java.util.function`)
| Interface | Signature | Use |
|---|---|---|
| `Supplier<T>` | `() → T` | Lazy values, factories (`orElseGet`) |
| `Consumer<T>` | `T → void` | `forEach` |
| `BiConsumer<T,U>` | `(T,U) → void` | `map.forEach` |
| `Function<T,R>` | `T → R` | `map` (`andThen`, `compose`) |
| `BiFunction<T,U,R>` | `(T,U) → R` | `map.merge`, `reduce` |
| `Predicate<T>` | `T → boolean` | `filter` (`and`, `or`, `negate`) |
| `UnaryOperator<T>` / `BinaryOperator<T>` | `T → T` / `(T,T) → T` | `replaceAll` / `reduce` |
Primitive variants (`IntPredicate`, `ToIntFunction`, …) avoid boxing. Custom: annotate with `@FunctionalInterface`.

## 3. Streams
```
source → intermediate ops (lazy) → terminal op (triggers execution)
```
```java
record Employee(String name, String dept, double salary, int age) {}

List<String> names = employees.stream()
        .filter(e -> e.salary() > 50_000)          // intermediate
        .sorted(Comparator.comparingDouble(Employee::salary).reversed())
        .map(Employee::name)
        .limit(3)
        .toList();                                 // terminal (Java 16; unmodifiable)
```
- **Lazy**: nothing runs until a terminal operation. **Vertical** processing (each element flows through the whole pipeline); short-circuiting ops (`limit`, `findFirst`, `anyMatch`) stop early.
- A stream can be consumed **only once** (`IllegalStateException` otherwise).
- Streams don't modify the source. Keep lambdas **stateless and side-effect free**.

| Intermediate | Terminal |
|---|---|
| `filter`, `map`, `flatMap`, `distinct`, `sorted`, `peek`, `limit`, `skip`, `mapToInt`, `boxed`, `takeWhile`/`dropWhile` (9) | `forEach`, `collect`, `toList`, `reduce`, `count`, `min`/`max`, `anyMatch`/`allMatch`/`noneMatch`, `findFirst`/`findAny`, `sum`/`average` (primitive streams) |

### `map` vs `flatMap`
```java
List<List<String>> nested = List.of(List.of("a", "b"), List.of("c"));
nested.stream().map(List::size).toList();               // [2, 1]: one output per input
nested.stream().flatMap(List::stream).toList();         // [a, b, c]: one-to-many, flattened
sentences.stream().flatMap(s -> Arrays.stream(s.split(" "))).distinct().toList();   // unique words
```

### Collectors: the interview favourites
```java
// group by department
Map<String, List<Employee>> byDept = employees.stream().collect(Collectors.groupingBy(Employee::dept));
// count per department
Map<String, Long> countByDept = employees.stream().collect(Collectors.groupingBy(Employee::dept, Collectors.counting()));
// average salary per dept
Map<String, Double> avg = employees.stream().collect(Collectors.groupingBy(Employee::dept, Collectors.averagingDouble(Employee::salary)));
// highest paid per dept
Map<String, Optional<Employee>> top = employees.stream()
        .collect(Collectors.groupingBy(Employee::dept, Collectors.maxBy(Comparator.comparingDouble(Employee::salary))));
// partition
Map<Boolean, List<Employee>> seniors = employees.stream().collect(Collectors.partitioningBy(e -> e.age() > 40));
// to map (⚠️ duplicate keys throw IllegalStateException unless you pass a merge function)
Map<String, Double> salaryByName = employees.stream()
        .collect(Collectors.toMap(Employee::name, Employee::salary, (a, b) -> a));
// joining
String csv = employees.stream().map(Employee::name).collect(Collectors.joining(", ", "[", "]"));
// sorted map
Map<String, Long> sorted = employees.stream().collect(Collectors.groupingBy(Employee::dept, TreeMap::new, Collectors.counting()));
```

### Classic coding questions
```java
// character frequency
Map<Character, Long> freq = s.chars().mapToObj(c -> (char) c)
        .collect(Collectors.groupingBy(Function.identity(), LinkedHashMap::new, Collectors.counting()));
// first non-repeated character
Character first = freq.entrySet().stream().filter(e -> e.getValue() == 1)
        .map(Map.Entry::getKey).findFirst().orElse(null);
// second highest salary
Optional<Double> second = employees.stream().map(Employee::salary).distinct()
        .sorted(Comparator.reverseOrder()).skip(1).findFirst();
// duplicates in a list
Set<Integer> seen = new HashSet<>();
List<Integer> dups = nums.stream().filter(n -> !seen.add(n)).distinct().toList();
// sum of squares of evens
int sum = nums.stream().filter(n -> n % 2 == 0).mapToInt(n -> n * n).sum();
// reduce
int product = nums.stream().reduce(1, (a, b) -> a * b);
```
Primitive streams: `IntStream.range(0, n)`, `rangeClosed`, `mapToInt(...).sum()/average()/summaryStatistics()`, which avoid boxing.

### Parallel streams
`list.parallelStream()` uses the common **ForkJoinPool** (`#cores - 1` threads).
Use only for large, CPU-bound, stateless, easily-splittable work (ArrayList, arrays; not LinkedList or I/O). **Never** do blocking I/O there or in a web server request path, and don't mutate shared state (`forEach(list::add)` races). Order: `forEachOrdered`.

## 4. Optional
A container for a value that **may be absent**, meant for **return types**.
```java
Optional<User> u = userRepository.findById(id);
String name = u.map(User::getName).orElse("unknown");
User user   = u.orElseThrow(() -> new ResourceNotFoundException("User " + id));
u.ifPresentOrElse(this::notify, () -> log.warn("missing"));
Optional.ofNullable(maybeNull).filter(s -> !s.isBlank()).map(String::trim);
```
| Method | Note |
|---|---|
| `of(x)` | NPE if x is null |
| `ofNullable(x)` | empty if null |
| `orElse(v)` | **v is evaluated even when present** |
| `orElseGet(supplier)` | Lazy; use for expensive defaults |
| `orElseThrow()` | Java 10; preferred over `get()` |
| `map` / `flatMap` / `filter` / `or` (9) / `stream` (9) | Chaining |
**Don't:** use Optional for fields, method parameters, or collections (return an empty list), call `get()` without checking, or write `if (opt.isPresent()) opt.get()` (use `map`/`orElse`). `Optional` isn't `Serializable`.

## 5. Other Java 8 additions
- **Default and static interface methods** ([52](52_OOP%20in%20Java%20-%20Classes%2C%20Interfaces%2C%20Inheritance%20and%20Polymorphism.md)).
- **`java.time`**: immutable and thread-safe `LocalDate`, `LocalDateTime`, `ZonedDateTime`, `Instant`, `Duration`, `Period`, `DateTimeFormatter`. Replaces the mutable, broken `Date`/`Calendar`/`SimpleDateFormat` (not thread-safe!).
- `CompletableFuture` ([59](59_Multithreading%20Basics%20-%20Threads%2C%20Executors%20and%20CompletableFuture.md)), `ConcurrentHashMap` improvements, `StringJoiner`, `Base64`.

---

## Interview Q&A
**Q1. What is a functional interface? Name some.**
An interface with exactly one abstract method: `Runnable`, `Callable`, `Comparator`, `Supplier`, `Consumer`, `Function`, `Predicate`.

**Q2. Intermediate vs terminal operations? What does lazy mean?**
Intermediate ops return streams and don't execute; the terminal op triggers the pipeline, and elements are processed only as needed.

**Q3. `map` vs `flatMap`?**
One-to-one transformation vs one-to-many plus flattening (stream of streams → stream).

**Q4. Can a stream be reused?**
No. It's consumed once; create a new one.

**Q5. `orElse` vs `orElseGet`?**
`orElse` always evaluates its argument; `orElseGet` calls the supplier only when empty.

**Q6. When are parallel streams a bad idea?**
Small data, I/O or blocking work, order-sensitive or stateful ops, shared mutable state, server request threads (common pool contention).

**Q7. Why must captured variables be effectively final?**
Lambdas capture values (copies), not variables. Allowing mutation would be confusing and thread-unsafe.

**Q8. `Collectors.toMap` with duplicate keys?**
`IllegalStateException`, unless you supply a merge function.

**Q9. Stream vs for-loop: which is faster?**
Loops are often slightly faster for simple work. Streams win on readability and composability; measure before optimizing.

## Quick revision
- Lambda = implementation of a functional interface; captured vars effectively final; method refs.
- Supplier / Consumer / Function / Predicate / Unary-BinaryOperator.
- Stream = lazy pipeline, single use, terminal triggers; `map` vs `flatMap`.
- `groupingBy` (+`counting`/`averaging`/`maxBy`), `partitioningBy`, `toMap` (merge fn), `joining`.
- Optional for return values: `map`/`orElseGet`/`orElseThrow`, never `get()` blindly.
- Parallel streams: CPU-bound, big data only.
