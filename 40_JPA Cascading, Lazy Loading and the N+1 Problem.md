# JPA Cascading, Lazy Loading and the N+1 Problem

> **Source:** Lecture 31 (`Notes_Of_Aditya/Lecture 31/code/jpaRelationshipDemo`)
> **Prerequisite:** [39_JPA Relationships](39_JPA%20Relationships.md) · **Next:** [41_Spring Data JPA](41_Spring%20Data%20JPA%20-%20Repositories%2C%20Queries%2C%20Sorting%20and%20Pagination.md)

## Why this topic matters
"What is the N+1 problem and how do you fix it?" is one of the most common JPA interview questions. Cascade and fetch type are the two settings that most often cause silent bugs and slow pages.

---

## 1. Cascading
A **cascade** propagates an EntityManager operation from one entity to related entities.
```java
@Entity @Table(name = "orders")
public class Order {
    @OneToMany(mappedBy = "order", cascade = CascadeType.PERSIST)
    private List<OrderItem> items = new ArrayList<>();

    public void addItem(OrderItem item) { items.add(item); item.setOrder(this); }   // sync both sides
}
@Entity
public class OrderItem {
    @ManyToOne @JoinColumn(name = "order_id")     // OWNING side (FK)
    private Order order;
}

order.addItem(i1); order.addItem(i2);
entityManager.persist(order);     // → persist(order), persist(i1), persist(i2)
```
| CascadeType | Propagates |
|---|---|
| `PERSIST` | `persist()` of new children |
| `MERGE` | `merge()` of detached children |
| `REMOVE` | `remove()`, deleting children (children deleted **before** the parent to respect the FK) |
| `REFRESH` / `DETACH` | `refresh()` / `detach()` |
| `ALL` | all of the above |

- **No cascade by default.** Without it, new children must be persisted separately, or you get `TransientPropertyValueException`.
- **Cascade ≠ ownership.** Ownership decides who writes the FK; cascade decides lifecycle propagation.
- Use `REMOVE`/`ALL` only when the child **belongs exclusively** to the parent (`OrderItem`). **Never on shared relations** like `@ManyToMany Set<Course>`, because deleting a student would delete the course for everyone.
- `orphanRemoval = true`: removing a child **from the collection** deletes its row (cascade REMOVE only fires when the parent is removed).

## 2. Fetch strategies
| | Meaning | Spec status |
|---|---|---|
| `FetchType.EAGER` | Load with the owner | **Requirement** |
| `FetchType.LAZY` | Load on first access | **Hint** (Hibernate honours it via proxies / lazy collections) |

| Annotation | Default |
|---|---|
| `@ManyToOne`, `@OneToOne` | **EAGER** |
| `@OneToMany`, `@ManyToMany` | **LAZY** |
Best practice: make everything **LAZY** and fetch what each use case needs **per query**.

### How lazy loading works (lecture code)
```java
@Entity
public class Student {
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "dept_id")    private Department department;
    @OneToOne(fetch = FetchType.LAZY)  @JoinColumn(name = "profile_id") private Profile profile;
}

public Student fetchStudentById(Long id) {
    Student s1 = studentRepository.findById(id);          // SELECT student only; department = uninitialized PROXY
    System.out.println("Lazily fetched Student");
    Department d1 = s1.getDepartment();                    // still a proxy (getting the reference alone ≠ load)
    System.out.println("Lazily fetched Department");
    Profile p1 = s1.getProfile();
    System.out.println("Lazily fetched Profile");
    return s1;
}
```
SQL runs only when **real state** is accessed (`d1.getName()`). Calling `getId()` on a proxy usually doesn't hit the DB.

### ⚠️ `LazyInitializationException` and Open-In-View
Accessing a lazy association **after the persistence context is closed** throws `LazyInitializationException: could not initialize proxy - no Session`.
In Spring Boot web apps, `spring.jpa.open-in-view=true` (**default**, with a startup warning) keeps the EntityManager open for the whole HTTP request, so lazy loads work during JSON serialization. That hides N+1 queries in the view layer and holds DB connections longer. Many teams set it to `false` and fetch explicitly.

## 3. The N+1 problem
```java
List<Student> students = studentRepository.findAll();       // 1 query
for (Student s : students)
    System.out.println(s.getDepartment().getName());          // +1 query PER student (lazy proxy init)
```
```sql
SELECT id, name, dept_id FROM student;            -- 1
SELECT * FROM department WHERE id = 1;            -- +N
SELECT * FROM department WHERE id = 2;
...
```
Collection version: `findAll()` departments, then `d.getStudents()` for each department means 1 + N.
- The first-level cache can reduce N (100 students in 3 departments gives 1 + 3), but the **fetch-plan problem remains**.
- **EAGER is not the fix.** It applies globally (even when you don't need it) and Hibernate may still issue secondary selects.

### Detect it
```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE      # parameter values
```
Look for the **same SELECT repeating inside a loop**.

### Fix it: fetch what the use case needs
```java
// 1) Entity graph (Spring Data JPA repository interface)
@EntityGraph(attributePaths = {"department", "profile"})
List<Student> findAll();

// 2) JPQL fetch join
@Query("SELECT s FROM Student s JOIN FETCH s.department")
List<Student> findAllWithDepartment();

// 3) Batch fetching (loads proxies in IN(...) batches)
spring.jpa.properties.hibernate.default_batch_fetch_size=50
// or @BatchSize(size = 50) on the association/collection

// 4) DTO projection — select only needed columns
@Query("SELECT new com.x.StudentView(s.name, d.name) FROM Student s JOIN s.department d")
```
⚠️ Fetch-joining **multiple collections** produces a cartesian product (`MultipleBagFetchException` with Lists). Fetch one collection per query or use batch fetching. Fetch join + pagination on a collection makes Hibernate paginate **in memory** (warning HHH90003004).

## ⚠️ Issues in the lecture code (✅ verified by compiling)
1. **`StudentRepository.findAll()` has an empty body**, so the project **doesn't compile** (`missing return statement`, confirmed with `mvn compile`). Also, `@EntityGraph` only works on **Spring Data repository interfaces**. On a hand-written `@Repository` class with `EntityManager`, it does nothing. Use `entityManager.createQuery("SELECT s FROM Student s JOIN FETCH s.department", Student.class)` or an `EntityGraph` hint instead.
2. `fetchStudentById` isn't `@Transactional`. It only works thanks to open-in-view, and returning the entity serializes **Hibernate proxies** with Jackson (fails with `hibernateLazyInitializer`/`ByteBuddyInterceptor` errors unless handled). Return a DTO.
3. `removeDepartment(id)`: a department still referenced by students fails with a **FK constraint violation** (no cascade or orphan handling, and the inverse side is commented out). It also NPEs on an unknown id.
4. Build gotcha: on **JDK 23+**, annotation processing is off by default, so Lombok getters/setters "disappear" (`cannot find symbol`). Add Lombok to `maven-compiler-plugin` → `annotationProcessorPaths` (Spring Initializr does this) and use a Lombok version that supports your JDK.

---

## Interview Q&A
**Q1. What is the N+1 problem?**
One query loads N parents, then accessing a lazy association triggers one extra query per parent: N+1 round trips.

**Q2. How do you fix N+1?**
Fetch join (`JOIN FETCH`), `@EntityGraph`, batch fetching (`default_batch_fetch_size`/`@BatchSize`), or DTO projections. Not by switching to EAGER.

**Q3. Default fetch types?**
`*ToOne` are EAGER; `*ToMany` are LAZY.

**Q4. Why not make everything EAGER?**
It loads unneeded graphs on every query and can still cause N+1 via secondary selects.

**Q5. What is `LazyInitializationException`?**
Accessing an uninitialized lazy proxy or collection after its persistence context closed.

**Q6. What is open-in-view? Pros and cons?**
It keeps the EntityManager open during the web request (default true in Boot). Convenient, but it hides lazy loading in controllers/serialization, causes N+1, and holds connections longer.

**Q7. `CascadeType.REMOVE` vs `orphanRemoval`?**
REMOVE deletes children when the parent is deleted. orphanRemoval also deletes a child when it's removed from the parent's collection.

**Q8. When is `CascadeType.ALL` appropriate?**
When the child's lifecycle is fully owned by the parent (composition), e.g. Order → OrderItems. Never for shared entities.

## Quick revision
- Cascade = lifecycle propagation (not ownership); none by default; REMOVE only for owned children.
- LAZY everywhere; fetch per use case (entity graph, JOIN FETCH, batch size, DTO).
- N+1 = 1 parent query + N association queries; detect with SQL logs.
- EAGER isn't a fix; watch open-in-view and `LazyInitializationException`.
