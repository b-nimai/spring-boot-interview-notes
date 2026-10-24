# equals, hashCode, Immutability and Records

> **Part of:** Core Java (51–62) · **Related:** [55_HashMap Internals](55_HashMap%20Internals.md), [39_JPA Relationships](39_JPA%20Relationships.md) (entity equals)

## Why this topic matters
"Why override hashCode when you override equals?" is one of the most asked Java questions. Immutability and records are equally common, and they matter for DTOs, map keys, and thread safety.

---

## 1. The `equals` contract
For non-null `x, y, z`:
| Property | Rule |
|---|---|
| Reflexive | `x.equals(x)` is true |
| Symmetric | `x.equals(y)` ⇔ `y.equals(x)` |
| Transitive | x=y and y=z ⇒ x=z |
| Consistent | Same result while the compared fields don't change |
| Non-null | `x.equals(null)` is false |

## 2. The `hashCode` contract
1. Equal objects **must** have equal hash codes.
2. Unequal objects *may* share a hash code (collisions are allowed, but hurt performance).
3. Consistent while the object's fields don't change.

### Why override both
`HashMap`/`HashSet` first find the **bucket via `hashCode()`**, then compare with `equals()`. Override only `equals` and two "equal" objects land in different buckets:
```java
class Point { int x, y; /* equals overridden, hashCode NOT */ }
Set<Point> set = new HashSet<>();
set.add(new Point(1, 2));
set.contains(new Point(1, 2));   // false! different identity hash → different bucket
```

## 3. Correct implementation
```java
public final class Point {
    private final int x, y;
    public Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point p)) return false;     // or getClass() != o.getClass() for strict class equality
        return x == p.x && y == p.y;
    }
    @Override
    public int hashCode() { return Objects.hash(x, y); }   // use the SAME fields as equals
    @Override
    public String toString() { return "Point[x=" + x + ", y=" + y + "]"; }
}
```
- Use `Objects.equals(a, b)` for nullable fields. Use `Double.compare` for doubles and `Arrays.equals` for arrays.
- `instanceof` vs `getClass()`: `instanceof` allows subclasses but can break symmetry if the subclass adds fields; `getClass()` is strict.
- `Comparable.compareTo` should be **consistent with equals** (`TreeSet` uses `compareTo`, not `equals`).

### ⚠️ Mutable keys
```java
Point p = new MutablePoint(1, 2);
map.put(p, "A");
p.setX(5);              // hash changed → entry is now "lost" in the wrong bucket
map.get(p);             // null
```
**Map keys and set elements should be immutable** (or at least their hashed fields).

### JPA entity equality (link to note 39)
Don't use `@Data` on entities. Base equality on the id carefully: a new entity's id is null, so hashing on it breaks `HashSet` after persist. A common pattern is `equals` by id when non-null, and `hashCode` returning `getClass().hashCode()`, or use a natural business key.

## 4. Immutability
An immutable object's state can't change after construction. Recipe:
1. Declare the class `final` (or use a private constructor + factory), so subclasses can't add mutability.
2. Make all fields `private final`.
3. Provide no setters.
4. **Defensively copy** mutable inputs and outputs.
5. Don't leak `this` during construction.
```java
public final class Order {
    private final String id;
    private final List<String> items;
    public Order(String id, List<String> items) {
        this.id = id;
        this.items = List.copyOf(items);        // defensive copy (also rejects nulls) → unmodifiable
    }
    public List<String> getItems() { return items; }       // already unmodifiable → safe to return
    public Order withItem(String item) {                   // "wither": returns a NEW object
        List<String> copy = new ArrayList<>(items); copy.add(item);
        return new Order(id, copy);
    }
}
```
**Benefits:** thread-safe without locks, safe as map keys, hash can be cached, easy reasoning and sharing. **Cost:** extra allocations (usually negligible).
Examples in the JDK: `String`, wrappers, `BigDecimal`, `LocalDate`/`java.time`, `List.of()`.

### Unmodifiable ≠ immutable
```java
List<String> base = new ArrayList<>(List.of("a"));
List<String> view = Collections.unmodifiableList(base);   // a VIEW
base.add("b");                                              // view now shows "b"!
List<String> snapshot = List.copyOf(base);                  // true immutable copy
```
`List.of`/`Map.of`/`Set.of` (Java 9): immutable, **null elements throw NPE**, `add` throws `UnsupportedOperationException`.

## 5. Records (Java 16)
A concise **immutable data carrier**:
```java
public record UserDto(Long id, String name, String email) {
    public UserDto {                                    // compact constructor → validation
        Objects.requireNonNull(name);
        if (email != null && !email.contains("@")) throw new IllegalArgumentException("bad email");
    }
    public String displayName() { return name.toUpperCase(); }   // extra methods allowed
    public static UserDto of(String name) { return new UserDto(null, name, null); }
}
// auto-generated: private final fields, canonical constructor, accessors id()/name()/email(),
// equals(), hashCode(), toString()
```
- Implicitly `final`; can't extend classes (implicitly extends `java.lang.Record`); **can implement interfaces**.
- Fields are final, but **shallowly** immutable: a `List` component can still be mutated, so copy it in the compact constructor (`items = List.copyOf(items);`).
- Great for DTOs, API requests/responses, value objects, map keys, and Spring `@ConfigurationProperties` (constructor binding).
- **Not** for JPA entities (entities need a no-arg constructor, mutability, and proxies).

> 🟦 TS: a record is like an `interface`/`type` plus a class with `readonly` fields, but with real structural `equals`/`hashCode` (TS has no value equality).

## 6. Lombok equivalents (common in Spring code)
| Lombok | Generates |
|---|---|
| `@Getter`/`@Setter` | accessors |
| `@EqualsAndHashCode` | equals/hashCode (`@EqualsAndHashCode.Exclude` for relations) |
| `@ToString` | toString (exclude relations) |
| `@Data` | all of the above + `@RequiredArgsConstructor` (avoid on entities) |
| `@Value` | immutable class (final fields, getters, equals/hashCode) |
| `@Builder` | builder pattern |
Records replace most `@Value`/DTO use cases.

---

## Interview Q&A
**Q1. Why must you override hashCode when you override equals?**
Hash-based collections locate the bucket by hashCode. Equal objects with different hashes break `HashMap`/`HashSet` lookups.

**Q2. Can two unequal objects have the same hashCode?**
Yes (a collision). The reverse isn't allowed: equal objects must share a hash.

**Q3. What happens if you mutate a HashMap key after insertion?**
Its hash changes, so `get`/`remove` look in the wrong bucket and the entry is effectively lost (a memory leak).

**Q4. How do you make a class immutable?**
A final class, private final fields, no setters, defensive copies in and out, and no `this` escape.

**Q5. Why are immutable objects thread-safe?**
Their state never changes after safe publication (final-field semantics), so there are no data races.

**Q6. `Collections.unmodifiableList` vs `List.copyOf`?**
A read-only view of the original (it reflects changes) vs an independent immutable copy.

**Q7. What does a record generate? Can a record be mutable?**
Private final fields, the canonical constructor, accessors, `equals`/`hashCode`/`toString`. It's shallowly immutable: mutable components can still change unless copied.

**Q8. Why not use records as JPA entities?**
JPA needs a no-arg constructor, mutable state, and proxy subclassing. Records are final and immutable.

## Quick revision
- equals: reflexive, symmetric, transitive, consistent, non-null. hashCode: equal ⇒ same hash.
- Override both, using the same fields (`Objects.equals`/`Objects.hash`).
- Immutable keys; mutating a key loses the entry.
- Immutable recipe: final class + private final fields + no setters + defensive copies.
- Records = concise immutable DTOs (shallow); not entities.
