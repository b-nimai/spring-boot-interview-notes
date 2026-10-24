# Collections Framework

> **Part of:** Core Java (51–62) · **Next:** [55_HashMap Internals](55_HashMap%20Internals.md)

## Why this topic matters
You'll use collections in every DSA and LLD round. Interviewers ask which collection to pick and why, Big-O complexities, ArrayList vs LinkedList, fail-fast iterators, and Comparable vs Comparator.

---

## 1. Hierarchy
```
Iterable
 └─ Collection
     ├─ List   (ordered, index, duplicates)   ArrayList, LinkedList, (Vector, Stack: legacy)
     ├─ Set    (no duplicates)                HashSet, LinkedHashSet, TreeSet (SortedSet/NavigableSet)
     └─ Queue  (FIFO/priority)                PriorityQueue, ArrayDeque, LinkedList
          └─ Deque (both ends)                ArrayDeque, LinkedList
Map (NOT a Collection)                        HashMap, LinkedHashMap, TreeMap, (Hashtable: legacy)
```
> 🟦 TS: `Array` ≈ `ArrayList`, `Set` ≈ `LinkedHashSet` (JS Sets keep insertion order!), `Map` ≈ `LinkedHashMap`. Java's `HashSet`/`HashMap` have **no ordering guarantee**.

## 2. Lists
| | ArrayList | LinkedList |
|---|---|---|
| Structure | Dynamic array (grows ~1.5×) | Doubly linked list |
| `get(i)` | **O(1)** | O(n) |
| Add at end | O(1) amortized | O(1) |
| Add/remove in the middle | O(n) (shift) | O(1) *after* reaching the node (finding it is O(n)) |
| Memory | Compact, cache-friendly | Node objects (prev/next), extra overhead |
| Use | **Default choice** | Rarely; for a deque prefer `ArrayDeque` |
```java
List<Integer> list = new ArrayList<>(List.of(5, 3, 1));
list.remove(Integer.valueOf(3));   // remove(Object) vs remove(int index): a classic trap
```

## 3. Sets
| | HashSet | LinkedHashSet | TreeSet |
|---|---|---|---|
| Backed by | HashMap | LinkedHashMap | Red-black tree (TreeMap) |
| Order | None | **Insertion** | **Sorted** (natural/Comparator) |
| add/contains/remove | O(1) avg | O(1) avg | **O(log n)** |
| null | One null | One null | ❌ (with natural ordering) |
| Uniqueness by | `hashCode` + `equals` | same | `compareTo`/`compare` (== 0 means duplicate) |
TreeSet/TreeMap extras: `first()`, `last()`, `floor()`, `ceiling()`, `headSet()`, `tailSet()`, `subSet()`. Great for range queries in DSA.

## 4. Maps
| | HashMap | LinkedHashMap | TreeMap | ConcurrentHashMap |
|---|---|---|---|---|
| Order | None | Insertion (or **access** order) | Sorted by key | None |
| get/put | O(1) avg | O(1) avg | O(log n) | O(1) avg |
| null key/value | 1 null key, null values | same | ❌ null key | ❌ nulls |
| Thread-safe | ❌ | ❌ | ❌ | ✅ |
**LRU cache** in a few lines:
```java
class LRU<K, V> extends LinkedHashMap<K, V> {
    private final int cap;
    LRU(int cap) { super(16, 0.75f, true); this.cap = cap; }          // accessOrder = true
    @Override protected boolean removeEldestEntry(Map.Entry<K, V> e) { return size() > cap; }
}
```
Useful Map API (Java 8+):
```java
map.getOrDefault(k, 0);
map.putIfAbsent(k, v);
map.merge(word, 1, Integer::sum);                         // frequency count
map.computeIfAbsent(key, k -> new ArrayList<>()).add(v);  // group-by
map.entrySet().forEach(e -> ...); for (var e : map.entrySet()) ...
```

## 5. Queues and deques
```java
Queue<Integer> q = new ArrayDeque<>();   q.offer(1); q.poll(); q.peek();     // FIFO
Deque<Integer> stack = new ArrayDeque<>(); stack.push(1); stack.pop();      // LIFO (prefer over Stack)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();                      // smallest first
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);       // by 2nd element
```
| Operation | Throws on failure | Returns special value |
|---|---|---|
| Insert | `add` | `offer` (false) |
| Remove head | `remove` | `poll` (null) |
| Examine | `element` | `peek` (null) |
PriorityQueue: binary heap. offer/poll are O(log n), peek is O(1). **Iteration order isn't sorted.** `ArrayDeque` doesn't allow nulls.
⚠️ `(a, b) -> a - b` can overflow with large ints. Use `Integer.compare(a, b)`.

## 6. Comparable vs Comparator
| Comparable | Comparator |
|---|---|
| `compareTo(T o)` **inside** the class | `compare(T a, T b)` in a **separate** object |
| One **natural** order | Many custom orders |
| `Collections.sort(list)` | `list.sort(cmp)` |
```java
record Employee(String name, int age, double salary) implements Comparable<Employee> {
    public int compareTo(Employee o) { return Integer.compare(age, o.age); }   // natural: by age
}
employees.sort(Comparator.comparing(Employee::name));
employees.sort(Comparator.comparingDouble(Employee::salary).reversed()
                         .thenComparing(Employee::name));
employees.sort(Comparator.comparing(Employee::name, Comparator.nullsLast(Comparator.naturalOrder())));
```
Sorting: `List.sort`/`Collections.sort` use **TimSort** (stable, O(n log n)). `Arrays.sort(int[])` uses dual-pivot quicksort for primitives.

## 7. Iteration and fail-fast
```java
for (String s : list) if (s.isEmpty()) list.remove(s);    // ❌ ConcurrentModificationException
list.removeIf(String::isEmpty);                           // ✅
Iterator<String> it = list.iterator();
while (it.hasNext()) if (it.next().isEmpty()) it.remove(); // ✅ iterator's own remove
```
> ✅ **Verified trick question:** for `["a", "", "b"]`, removing `""` (the **second-to-last** element) inside a for-each does **not** throw. After the removal, `size` drops to 2 and the cursor is 2, so `hasNext()` returns false and the loop ends before the `modCount` check (and `"b"` is never visited). For `["", "a", "b"]` it throws `ConcurrentModificationException`.

- **Fail-fast** iterators (ArrayList, HashMap) check `modCount` and throw `ConcurrentModificationException` on structural modification. Best-effort, not a thread-safety guarantee.
- **Fail-safe / weakly consistent** iterators (`ConcurrentHashMap`, `CopyOnWriteArrayList`) work on a snapshot or tolerate concurrent changes and never throw CME.

## 8. Utility classes and gotchas
```java
Arrays.asList(1, 2, 3);          // FIXED-SIZE list backed by the array: set() ok, add() → UnsupportedOperationException
List.of(1, 2, 3);                // immutable, no nulls
Collections.unmodifiableList(l); // read-only VIEW
Collections.synchronizedList(l); // locks every call (iteration still needs a manual synchronized block)
Collections.emptyList(), singletonList(x), frequency(c, x), reverse(l), shuffle(l), max(c, cmp)
```
- Initial capacity: `new ArrayList<>(n)` / `new HashMap<>(expected / 0.75f + 1)` avoids resizes. Java 19+: `HashMap.newHashMap(n)`.
- Legacy synchronized classes (`Vector`, `Hashtable`, `Stack`): avoid. Use `ArrayList`, `ConcurrentHashMap`, `ArrayDeque`.

## 9. Picking the right collection
| Need | Choose |
|---|---|
| Indexed list | `ArrayList` |
| Unique, fast lookup | `HashSet` |
| Unique + insertion order | `LinkedHashSet` |
| Sorted / range queries | `TreeSet` / `TreeMap` |
| Key-value | `HashMap` (`LinkedHashMap` for order/LRU) |
| Stack / queue / deque | `ArrayDeque` |
| Top-K / scheduling | `PriorityQueue` |
| Concurrent map | `ConcurrentHashMap` |
| Read-heavy concurrent list | `CopyOnWriteArrayList` |

---

## Interview Q&A
**Q1. ArrayList vs LinkedList?**
Array-backed (O(1) random access, cache-friendly; the default choice) vs doubly linked (O(1) insert at a known node; poor locality).

**Q2. HashSet vs TreeSet vs LinkedHashSet?**
No order with O(1) ops / sorted with O(log n) / insertion order with O(1).

**Q3. How does HashSet ensure uniqueness?**
It's backed by a HashMap (elements are keys with a dummy value), so it relies on `hashCode` + `equals`.

**Q4. Comparable vs Comparator?**
A natural order inside the class vs external, multiple orders (composable with `thenComparing`).

**Q5. What is ConcurrentModificationException? How do you avoid it?**
Structural modification during fail-fast iteration. Use `Iterator.remove`, `removeIf`, or concurrent collections.

**Q6. Fail-fast vs fail-safe iterators?**
Throw CME on modification (modCount check) vs iterate over a snapshot or weakly consistent view.

**Q7. `Arrays.asList` vs `List.of`?**
A fixed-size view backed by the array (`set` allowed, nulls allowed) vs an immutable list (no nulls, no modification).

**Q8. How do you implement an LRU cache in Java?**
`LinkedHashMap` with `accessOrder = true` + `removeEldestEntry`. Or a HashMap + doubly linked list by hand (an LLD favourite).

## Quick revision
- List (ArrayList default) / Set (Hash, Linked, Tree) / Queue (ArrayDeque, PriorityQueue) / Map (Hash, Linked, Tree, Concurrent).
- O(1) hash vs O(log n) tree; LinkedHash = order.
- Comparable (natural) vs Comparator (custom, chainable); TimSort is stable.
- Don't modify while iterating; use `removeIf`/Iterator.
- `Arrays.asList` is fixed-size; `List.of` is immutable with no nulls.
