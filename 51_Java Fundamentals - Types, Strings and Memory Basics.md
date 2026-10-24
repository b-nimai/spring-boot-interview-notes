# Java Fundamentals: Types, Strings and Memory Basics

> **Part of:** Core Java for interviews (51–62) · **Coming from TypeScript?** Watch the 🟦 TS boxes.

## Why this topic matters
Product-company interviews open with fundamentals: primitive vs wrapper, `==` vs `equals`, String immutability and the pool, pass-by-value, `static`/`final`. These are easy points to win or lose.

---

## 1. Compilation and platform
```
Hello.java --javac--> Hello.class (bytecode) --JVM (interpreter + JIT)--> native code
```
- **JDK** = JRE + compiler and tools. **JRE** = JVM + libraries. **JVM** runs bytecode ("write once, run anywhere").
- **JIT** (HotSpot) compiles hot methods to native code at runtime, which is why Java "warms up".
> 🟦 TS: `tsc` erases types to JS. Java keeps types in bytecode and the JVM verifies them. Generics are the exception (erased, see [56](56_Generics.md)).

## 2. Primitive types vs reference types
| Primitive | Size | Default (fields) | Wrapper |
|---|---|---|---|
| `byte` | 8-bit | 0 | `Byte` |
| `short` | 16-bit | 0 | `Short` |
| `int` | 32-bit | 0 | `Integer` |
| `long` | 64-bit | 0L | `Long` |
| `float` | 32-bit | 0.0f | `Float` |
| `double` | 64-bit | 0.0d | `Double` |
| `char` | 16-bit UTF-16 | `'\u0000'` | `Character` |
| `boolean` | JVM-dependent | false | `Boolean` |
- **Primitives** hold the value directly: no null, no methods, fast.
- **References** point to objects on the heap and can be `null`.
- **Local variables have no default** and must be assigned before use (compile error otherwise).
> 🟦 TS has one `number` (64-bit float). In Java `int` overflows silently: `Integer.MAX_VALUE + 1 == Integer.MIN_VALUE`. Use `long` or `Math.addExact` (throws on overflow). Money → `BigDecimal`, never `double` (`0.1 + 0.2 != 0.3`).

### Autoboxing pitfalls
```java
Integer a = 127, b = 127;  a == b   // true  (Integer cache -128..127)
Integer c = 128, d = 128;  c == d   // false (different objects!) → always use equals()
Integer n = null; int x = n;        // NullPointerException (auto-unboxing null)
```
Integer division: `5 / 2 == 2`. Casting: `(int) 3.9 == 3` (truncates). Widening is implicit (`int → long`); narrowing needs a cast.

## 3. `==` vs `equals()`
| | `==` | `equals()` |
|---|---|---|
| Primitives | Value comparison | n/a |
| Objects | **Same reference?** | **Logically equal?** (if overridden) |
`Object.equals` defaults to `==`. String, wrappers, and collections override it. See [53](53_equals%2C%20hashCode%2C%20Immutability%20and%20Records.md).

## 4. Strings
### Immutability
```java
String s = "hello";
s.toUpperCase();           // returns a NEW string; s is still "hello"
s = s.concat(" world");    // s now points to a new object
```
Why immutable: **security** (class names, URLs, credentials can't change after checks), **thread safety**, **hash caching** (great as HashMap keys), and the **String pool**.

### String pool (interning)
```java
String a = "java";              // literal → pool
String b = "java";              // same pooled object
String c = new String("java");  // new heap object (plus the pooled literal)
a == b          // true
a == c          // false
a.equals(c)     // true
a == c.intern() // true  (intern returns the pooled instance)
```
The pool lives in the **heap** (moved from PermGen in Java 7). Compile-time constants are folded: `"ja" + "va" == "java"` is true; with a variable operand it's false.

### StringBuilder vs StringBuffer
| | String | StringBuilder | StringBuffer |
|---|---|---|---|
| Mutable | ❌ | ✅ | ✅ |
| Thread-safe | ✅ (immutable) | ❌ | ✅ (synchronized, slower) |
| Use | Constants, keys | String building in loops | Legacy shared use |
```java
String r = ""; for (...) r += x;          // O(n²): new String each iteration
StringBuilder sb = new StringBuilder(); for (...) sb.append(x);   // ✅
```
Useful: `isBlank()`, `strip()`, `repeat()`, `lines()` (Java 11), text blocks `"""…"""` (15), `String.join`, `String.format`/`formatted()`.

## 5. Pass-by-value (always)
Java is **always pass-by-value**. For objects, the *value of the reference* is copied.
```java
void change(StringBuilder sb) { sb.append("!"); }      // mutates the shared object → visible to caller
void reassign(StringBuilder sb) { sb = new StringBuilder("x"); }   // rebinds the local copy → caller unaffected
void inc(int x) { x++; }                                // caller's int unchanged
```
> 🟦 Same as JS/TS: objects are "shared by reference value"; reassigning a parameter never affects the caller.

## 6. Stack vs heap (basics)
| Stack (per thread) | Heap (shared) |
|---|---|
| Method frames, local primitives, references | All objects and arrays |
| LIFO, auto-freed when the method returns | Managed by the GC |
| `StackOverflowError` (deep recursion) | `OutOfMemoryError: Java heap space` |
Details: [61_JVM Memory and GC](61_JVM%20Memory%2C%20Class%20Loading%20and%20Garbage%20Collection.md).

## 7. Keywords that come up constantly
### `static`
Belongs to the **class**, not instances. Static fields are shared (one copy); static methods can't use `this` or instance members; static blocks run once at class load; static nested classes don't hold an outer reference.
### `final`
| On | Meaning |
|---|---|
| Variable | Assigned once (a **final reference ≠ immutable object**: a `final List` can still be modified) |
| Method | Can't be overridden |
| Class | Can't be extended (`String`, wrappers) |
`finally` = a block that always runs; `finalize()` = deprecated GC hook (don't use).
### Access modifiers
| Modifier | Class | Package | Subclass | World |
|---|---|---|---|---|
| `private` | ✅ | ❌ | ❌ | ❌ |
| *(default / package-private)* | ✅ | ✅ | ❌ (outside package) | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |
> 🟦 TS has no package-private. Java's default (no modifier) is **package-private**, not public.

### `var` (Java 10)
Local type inference only: `var list = new ArrayList<String>();`. Still statically typed; not allowed for fields or parameters.

## 8. `main` and miscellany
`public static void main(String[] args)`: public (JVM calls it), static (no instance yet), void. Since Java 21 (preview) and 25, simpler `void main()` forms exist.
`switch` expressions (14): `int len = switch (day) { case MON, FRI -> 6; default -> 0; };`

---

## Interview Q&A
**Q1. Why is String immutable?**
Security, thread safety, hashCode caching, and String-pool sharing. Changing a pooled string would affect every reference.

**Q2. How many objects does `new String("abc")` create?**
Up to two: the pooled literal `"abc"` (if not already pooled) and a new heap String.

**Q3. Is Java pass-by-value or pass-by-reference?**
Always pass-by-value. Object references are copied, so mutations are visible but reassignment isn't.

**Q4. `==` vs `equals`?**
Reference identity vs logical equality. Always use `equals` for objects (beware the `Integer` cache).

**Q5. StringBuilder vs StringBuffer?**
Both mutable. StringBuffer is synchronized (thread-safe, slower); StringBuilder is preferred.

**Q6. Can a `final` list be modified?**
Yes. `final` prevents reassignment, not mutation. Use `List.of` / `Collections.unmodifiableList` for an unmodifiable list.

**Q7. Why is `main` static?**
The JVM calls it before any object exists.

**Q8. What happens with `Integer.MAX_VALUE + 1`?**
Silent overflow to `Integer.MIN_VALUE`.

## Quick revision
- 8 primitives (no null) vs references (heap, nullable); autoboxing + the Integer cache (-128..127).
- `==` identity, `equals` value. Strings are immutable and pooled; StringBuilder for loops.
- Always pass-by-value. Stack = frames; heap = objects.
- `static` = class-level; `final` = no reassignment/override/extend; default access = package-private.
