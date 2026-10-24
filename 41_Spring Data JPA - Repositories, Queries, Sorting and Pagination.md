# Spring Data JPA: Repositories, Queries, Sorting and Pagination

> **Source:** Lecture 32 (`Notes_Of_Aditya/Lecture 32/code/springDataJPADemo`)
> **Related:** [19_Spring Data JPA basics](19_Spring%20Data%20JPA%20-%20Entity%2C%20Repository%20and%20Database%20Config.md) · [20_Derived Query Methods](20_Soft%20Delete%20and%20Derived%20Query%20Methods.md) · [40_N+1](40_JPA%20Cascading%2C%20Lazy%20Loading%20and%20the%20N%2B1%20Problem.md)

## Why this topic matters
Most Spring Boot apps use `JpaRepository`. Interviewers ask how an interface gets an implementation, how `save()` works, JPQL vs native queries, and Page vs Slice.

---

## 1. Where Spring Data fits
```
Service → Spring Data JPA repository (proxy) → EntityManager (JPA) → Hibernate → JDBC → DB
```
Spring Data is an umbrella project (JPA, JDBC, MongoDB, Redis, Elasticsearch, Cassandra, Neo4j…) with one idea: **declare a repository interface for a domain type, and the module supplies the implementation.**
| JPA | Spring Data JPA |
|---|---|
| Jakarta spec: `@Entity`, `EntityManager`, JPQL | Spring module: `JpaRepository`, derived queries, `@Query`, paging |
| Needs a provider (Hibernate) | Uses the configured provider underneath |
It reduces boilerplate but does **not** replace understanding transactions, entity states, fetching, or indexes.

## 2. Repository hierarchy
```
Repository<T,ID>                     (marker — expose only what you declare)
 ├─ CrudRepository → ListCrudRepository           (save, findById, findAll, existsById, count, delete…)
 └─ PagingAndSortingRepository → ListPaging…      (findAll(Sort), findAll(Pageable))
JpaRepository<T,ID> = ListCrud + ListPagingAndSorting + QueryByExampleExecutor
                      + flush, saveAndFlush, getReferenceById, deleteAllInBatch…
```
Since Spring Data 3, `PagingAndSortingRepository` **no longer extends** `CrudRepository`.
To expose a restricted API (e.g. no delete), extend `Repository<Student, Long>` and declare only the methods you want.

```java
public interface StudentRepository extends JpaRepository<Student, Long> { }   // no impl, no @Repository needed
```

## 3. How an interface becomes a bean: repository proxy
At startup Spring Data creates a **proxy** implementing your interface:
| Method kind | Handled by |
|---|---|
| Inherited CRUD (`save`, `findById`…) | `SimpleJpaRepository` (uses `EntityManager`) |
| Derived (`findByEmail`) | Query built from the method name |
| `@Query` | Your JPQL/SQL |
| `default` interface method | Java |
| Custom fragment (`StudentRepositoryCustom` + `Impl`) | Your class |
`SimpleJpaRepository` is `@Transactional(readOnly = true)` by default, with write methods overridden as `@Transactional`.

## 4. Key methods and their semantics
| Method | Notes |
|---|---|
| `save(e)` | **New → `persist`**, else **`merge`** (new = null id / version, or `Persistable.isNew()`). **Use the returned object.** |
| `saveAll(list)` | Calls `save` per entity. No automatic multi-row INSERT (batching needs config + non-IDENTITY ids) |
| `findById(id)` | `EntityManager.find` → `Optional` (persistence-context cache first) |
| `getReferenceById(id)` | Lazy proxy without a SELECT (great for setting FKs) |
| `findAll()` | **Everything**. Dangerous on big tables |
| `existsById`, `count` | Existence / count queries |
| `delete`, `deleteById` | Load + remove (lifecycle callbacks and cascades apply) |
| `deleteAllInBatch` | Single bulk DELETE: **skips** cascades, callbacks, and the persistence context |
| `flush` / `saveAndFlush` | Push SQL now (detect constraint errors early). **Not commit** |

### `save()` vs dirty checking
```java
@Transactional
public void rename(Long id, String name) {
    Student s = studentRepository.findById(id).orElseThrow();
    s.setName(name);                 // UPDATE at commit — no save() required
}
```
`save()` matters for new entities and for merging detached ones.

## 5. Derived query methods
`find|exists|count|delete` + `By` + predicate, e.g. `findTop5ByActiveTrueOrderByAgeDesc()`.
| Keyword | Example |
|---|---|
| `And`/`Or` | `findByNameAndAge` |
| `GreaterThan`/`LessThan`/`Between` | `findByAgeBetween(18, 25)` |
| `Like` (you supply `%`) vs `Containing` (adds `%…%`) | `findByNameLike("%har%")` vs `findByNameContaining("har")` |
| `StartingWith` / `EndingWith` | |
| `In` / `NotIn` | `findByAgeIn(List)` |
| `IsNull` / `IsNotNull`, `True` / `False` | |
| `IgnoreCase`, `OrderBy…Asc/Desc` | |
| `Top`/`First` | `findFirstByDepartmentNameOrderByAgeDesc` (always add an order!) |
Long names mean it's time for `@Query`, Specifications, Querydsl, or a custom fragment.

## 6. `@Query`: JPQL and native
```java
// JPQL — entity & field names, not tables/columns
@Query("SELECT s FROM Student s WHERE s.email = :email")
Optional<Student> findStudentByEmail(@Param("email") String email);

@Query("SELECT s FROM Student s JOIN s.department d WHERE d.active = true")   // join via relationship path
List<Student> findStudentsInActiveDepartments();

@Query("SELECT s.name, d.name FROM Student s LEFT JOIN s.department d")       // keeps students without dept
List<Object[]> names();                                                        // prefer DTO/interface projections

// Native SQL — table & column names (lecture code)
@Query(value = "select * from student where email = :email", nativeQuery = true)
Optional<Student> findByEmail(@Param("email") String email);
// newer: @NativeQuery("select * from student where email = :email")
```
- Positional params `?1, ?2` are **1-based**.
- **Write queries** (`UPDATE`/`DELETE`) need `@Modifying` + a transaction. They bypass the persistence context, so use `@Modifying(clearAutomatically = true)`.
- Native = vendor coupling; use it for DB-specific features or tuned SQL.

## 7. Sorting
```java
Sort sort = Sort.by("name").ascending().and(Sort.by("age").descending());
studentRepository.findAll(sort);
List<Student> findByActiveTrue(Sort sort);            // dynamic order on a derived query
```
Property names are **entity properties**. Never pass client input straight into `Sort`; validate it against an allow-list. Use `Sort.unsorted()`, not `null`.

## 8. Pagination
```java
Pageable pageable = PageRequest.of(0, 10, Sort.by("name").ascending().and(Sort.by("id")));   // page 0-based
Page<Student> page = studentRepository.findAll(pageable);
page.getContent(); page.getTotalElements(); page.getTotalPages(); page.hasNext(); page.isLast();
```
`offset = page × size` → `LIMIT 10 OFFSET 20` for page 2.
- Add a **unique tie-breaker** (`id`) to the sort, or rows can repeat or skip between pages.
- Cap the page size at the API boundary.
- A `Page` needs a **count query**. Supply `countQuery` for complex `@Query`s (without fetch joins or order by).

| `Page<T>` | `Slice<T>` |
|---|---|
| Content + `totalElements` + `totalPages` | Content + `hasNext` only |
| Extra `COUNT(*)` query | Fetches size+1 rows, no count |
| "Page 3 of 25" UI | Infinite scroll / "load more" |

Web shortcut: a controller parameter `Pageable pageable` binds `?page=0&size=10&sort=name,asc`.

## ⚠️ Issues in the lecture code
1. **`updateStudent` has no `@Transactional`.** `findById` runs in its own short transaction, so the returned entity is **detached** and the setters are **never saved** (no dirty checking, no `save()`). Add `@Transactional` (or call `save`).
2. `fetchAll(name)` builds a `Sort` but **never uses it** (`PageRequest.of(0, 2)` has no sort), and the `name` parameter is ignored. With no ORDER BY, page contents aren't deterministic.
3. `findByEmailLike(...)` returns `Optional<Student>`. If the pattern matches 2+ rows you get `IncorrectResultSizeDataAccessException`. Return a `List`.
4. `fetchStudentById` uses `Optional.get()` → `NoSuchElementException` → 500. Use `orElseThrow(() -> new ResourceNotFoundException(...))`.
5. The native query hard-codes the table name `student` (depends on the naming strategy). JPQL would avoid that coupling.

---

## Interview Q&A
**Q1. How does Spring Data create an implementation for an interface?**
It generates a runtime proxy. CRUD calls go to `SimpleJpaRepository`; query methods are parsed or prepared at startup.

**Q2. `CrudRepository` vs `JpaRepository`?**
JpaRepository adds List returns, paging and sorting, flush, `saveAndFlush`, batch deletes, and `getReferenceById`.

**Q3. How does `save()` decide insert vs update?**
`isNew` (null id/version or `Persistable`) → `persist`; otherwise `merge`. The returned instance is the managed one.

**Q4. JPQL vs native query?**
JPQL works on entities and fields and is portable. Native is raw SQL on tables and columns, vendor-specific, and may need result mapping.

**Q5. How do you write an update query?**
`@Modifying @Transactional @Query("UPDATE Student s SET s.active=false WHERE s.age > :a")`. Clear or refresh the persistence context afterwards.

**Q6. Page vs Slice?**
Page has totals (extra count query); Slice only knows whether there's a next chunk (cheaper).

**Q7. Why is `findAll()` dangerous?**
It loads every row into memory (DB, network, persistence context, JSON). Paginate.

**Q8. What's the problem with offset pagination on huge tables?**
Large OFFSETs scan and skip many rows. Use keyset/seek pagination (`WHERE id > :lastId ORDER BY id LIMIT n`) or Spring Data's `ScrollPosition`/`Window`.

## Quick revision
- Interface → proxy → `SimpleJpaRepository` / derived / `@Query`.
- `save` = persist or merge; managed entities update via dirty checking (needs `@Transactional`).
- Derived names for simple queries, `@Query` JPQL for the rest, native when needed, `@Modifying` for writes.
- `Sort` (validated) + `PageRequest.of(page, size, sort)`; Page = totals, Slice = hasNext.
