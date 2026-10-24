# HashMap Internals

> **Part of:** Core Java (51–62) · **Prerequisites:** [53_equals and hashCode](53_equals%2C%20hashCode%2C%20Immutability%20and%20Records.md), [54_Collections](54_Collections%20Framework.md)

## Why this topic matters
"How does HashMap work internally?" is probably *the* most frequently asked Java interview question at product companies, often followed by "what changed in Java 8?" and "how is ConcurrentHashMap different?".

---

## 1. Structure
```
table: Node<K,V>[]  (array of buckets, length = power of 2, default 16)
 [0] → null
 [1] → Node(hash, key, value, next) → Node → ...     (linked list on collision)
 [2] → TreeNode (red-black tree)                       (when a bucket gets large — Java 8+)
 ...
```
```java
static class Node<K,V> { final int hash; final K key; V value; Node<K,V> next; }
```
Defaults: **capacity 16**, **load factor 0.75**, threshold = capacity × LF = **12**.
The table is created **lazily** on the first `put`.

## 2. `put(key, value)` step by step
```
1. h = key.hashCode()
2. hash = h ^ (h >>> 16)            // "spread": mix high bits into low bits
3. index = (n - 1) & hash           // n = table length (power of 2) → fast modulo
4. bucket empty?  → place new Node
   else walk the bucket:
      same hash && (same ref || key.equals(existing)) → REPLACE value (return old)
      else append to list (or insert into tree)
5. list length ≥ 8 (TREEIFY_THRESHOLD) and table length ≥ 64 → convert bucket to red-black tree
   (if table < 64, resize instead)
6. ++size > threshold → resize()
```
- `null` key → hash 0 → bucket 0 (HashMap allows one null key).
- **Why `(n-1) & hash`?** With `n` a power of 2 it equals `hash % n` but is faster, and it only works because the capacity is always a power of 2 (`new HashMap<>(10)` rounds up to 16).
- **Why spread `h ^ (h >>> 16)`?** The index uses only the low bits; mixing in the high bits reduces collisions for hashes that differ only in the high bits.

## 3. `get(key)`
Compute the hash → index → check the first node → walk the list (O(k)) or tree (O(log k)), comparing `hash` then `==`/`equals`. Return the value or `null`.
⚠️ `get` returning `null` is ambiguous (absent, or a null value). Use `containsKey` or `getOrDefault`.

## 4. Collisions and the Java 8 treeification
| Bucket form | Lookup |
|---|---|
| Linked list | O(k) per bucket; worst case **O(n)** (all keys collide) |
| Red-black tree (Java 8+) | **O(log n)** worst case |
- Treeify when a bucket's list ≥ **8** and capacity ≥ **64**; untreeify back to a list at ≤ **6** (hysteresis avoids flip-flopping).
- Trees order by hash, then `compareTo` if the keys are `Comparable`, then a tie-breaker.
- Protects against **hash-flooding DoS** (attackers sending colliding keys).
- Java 8 also switched list insertion from **head** to **tail**, which fixed the infinite-loop bug of concurrent resizes in Java 7 (HashMap is still not thread-safe!).

## 5. Resize (rehash)
When `size > threshold`: **double** the capacity and redistribute.
Clever trick: because the capacity doubles, each node either **stays at index `i`** or moves to **`i + oldCap`**, decided by one bit (`hash & oldCap`). No full rehash computation, and relative order is preserved.
Resizing is O(n), so pre-size when you know the count: `new HashMap<>((int) (expected / 0.75f) + 1)` or `HashMap.newHashMap(expected)` (Java 19).

**Load factor trade-off:** lower = fewer collisions, more memory; higher = denser, more collisions. 0.75 balances the two.

## 6. Complexity
| Operation | Average | Worst (Java 8+) |
|---|---|---|
| put / get / remove / containsKey | **O(1)** | O(log n) (treeified bucket) |
| Iteration | O(capacity + size) | |
| Resize | O(n) (amortized away) | |
A bad `hashCode` (e.g. `return 42;`) degrades everything into a single bucket.

## 7. HashMap vs Hashtable vs ConcurrentHashMap vs synchronizedMap
| | HashMap | Hashtable (legacy) | `Collections.synchronizedMap` | **ConcurrentHashMap** |
|---|---|---|---|---|
| Thread-safe | ❌ | ✅ (every method synchronized) | ✅ (one mutex) | ✅ fine-grained |
| Nulls | 1 null key + null values | ❌ | per wrapped map | ❌ (ambiguity in concurrent `get`) |
| Iterator | Fail-fast | Fail-fast (Enumeration too) | Fail-fast (manual sync needed) | Weakly consistent, no CME |
| Performance under contention | — | Poor | Poor | High |

### ConcurrentHashMap internals
- **Java 7**: array of **Segments** (each a ReentrantLock + mini hash table); concurrency level 16.
- **Java 8+**: no segments. A `Node[]` table where an **empty bucket is filled with CAS** (lock-free) and a non-empty bucket is updated under **`synchronized` on the bucket's first node**. Reads are mostly **lock-free** (`volatile` reads). Treeification works as in HashMap. Resize is **cooperative** (multiple threads help transfer). Size uses `LongAdder`-style counter cells.
- Atomic compound ops: `putIfAbsent`, `computeIfAbsent`, `compute`, `merge`. Use these, not check-then-act:
```java
// ❌ race: two threads both see null
if (!map.containsKey(k)) map.put(k, v);
// ✅ atomic
map.putIfAbsent(k, v);
map.merge(word, 1, Integer::sum);               // thread-safe counter per key
map.computeIfAbsent(userId, id -> new ConcurrentLinkedQueue<>()).add(event);
```
⚠️ Don't recurse into the same map or do slow work inside `computeIfAbsent`'s function (it holds the bucket lock).

## 8. Related maps
- **LinkedHashMap**: HashMap + a doubly linked list through the entries (insertion or access order); LRU caches.
- **TreeMap**: red-black tree; sorted keys; O(log n); `floorKey`/`ceilingKey`; uses `compareTo`, not hashCode.
- **WeakHashMap**: keys weakly referenced (entries vanish when keys are GC'd).
- **IdentityHashMap**: compares keys with `==`.
- **EnumMap**: an array indexed by ordinal, very fast for enum keys.

---

## Interview Q&A
**Q1. Explain how HashMap works internally.**
An array of buckets. The hash is spread and indexed with `(n-1) & hash`; collisions chain in a linked list (tail insert) or a red-black tree once ≥ 8 nodes (capacity ≥ 64); `equals` decides a key match; it resizes ×2 at 75% load.

**Q2. What happens when two keys have the same hashCode?**
They go into the same bucket and are distinguished with `equals`. If `equals` is true the value is replaced; otherwise both are stored in the chain or tree.

**Q3. Why is the capacity a power of two?**
So `(n-1) & hash` replaces the costly modulo, and resizes can split buckets with a single bit check.

**Q4. What changed in Java 8?**
Treeification of large buckets (O(log n) worst case), tail insertion, a simpler hash spreading, and resize splitting by the `oldCap` bit.

**Q5. Why is HashMap not thread-safe? What can go wrong?**
Lost updates and inconsistent size during concurrent puts or resizes, plus an infinite loop on resize in Java 7. Use ConcurrentHashMap.

**Q6. How does ConcurrentHashMap achieve thread safety in Java 8?**
CAS for empty buckets, `synchronized` on the bucket head for updates, volatile reads (mostly lock-free gets), and cooperative resizing.

**Q7. Why doesn't ConcurrentHashMap allow null?**
`get` returning null couldn't distinguish "absent" from "mapped to null" without a racy `containsKey` check.

**Q8. What makes a good HashMap key?**
Immutable, with consistent and well-distributed `equals`/`hashCode` (String, Integer, records).

**Q9. Load factor and threshold?**
Threshold = capacity × load factor (16 × 0.75 = 12). Exceeding it triggers a resize to double the capacity.

## Quick revision
- `Node[]` buckets; default 16 / 0.75 → resize at 12; power-of-two capacity.
- index = `(n-1) & (h ^ h>>>16)`; collisions → list (tail) → tree at 8 (capacity ≥ 64), back at 6.
- Average O(1), worst O(log n) since Java 8.
- Resize doubles; nodes stay at `i` or move to `i+oldCap`.
- ConcurrentHashMap: CAS + per-bucket synchronized, no nulls, atomic `compute`/`merge`.
