# JPA Relationships: OneToOne, OneToMany, ManyToOne, ManyToMany

> **Source:** Lecture 30 (`Notes_Of_Aditya/Lecture 30/code/jpaRelationshipDemo`, uses Lombok)
> **Prerequisite:** [38_Hibernate Internals](38_Hibernate%20Internals%20-%20Persistence%20Context%2C%20Dirty%20Checking%20and%20Flush.md)

## Why this topic matters
"Owning side vs inverse side", "what does `mappedBy` mean", "how is many-to-many stored", and "why did my relationship not save?" are standard JPA interview questions. Getting them wrong leads to missing foreign keys, infinite JSON recursion, and N+1 queries.

---

## 1. References vs foreign keys
```
Java:  student.getDepartment()  → object reference
DB:    students.dept_id → departments.id   (foreign key)
```
Relationship mapping tells JPA how to translate one into the other.

## 2. Cardinality
| Annotation | Meaning | Example | DB representation |
|---|---|---|---|
| `@ManyToOne` | many → one | Student → Department | FK on the **many** table |
| `@OneToMany` | one → many | Department → Students | (same FK; inverse view) |
| `@OneToOne` | one ↔ one | User ↔ Profile | FK **+ UNIQUE** |
| `@ManyToMany` | many ↔ many | Student ↔ Course | **Join table** with two FKs |

`@OneToMany` and `@ManyToOne` are **two views of the same FK** (`students.dept_id`).

## 3. Unidirectional vs bidirectional
- **Unidirectional:** only one side has the field (`Student.department`).
- **Bidirectional:** both sides (`Student.department` + `Department.students`). **Still one FK in the DB.**
- The two Java fields **don't sync themselves**. Your code must keep both sides consistent.

## 4. Owning side vs inverse side
| | Owning side | Inverse side |
|---|---|---|
| Has | `@JoinColumn` / `@JoinTable` | `mappedBy = "fieldOnOwner"` |
| Controls the DB value? | ✅ **Yes**: JPA writes the FK/join row from here | ❌ No: navigation only |
| Typical | `@ManyToOne` side | `@OneToMany` side |

> Owning ≠ parent or "more important". It just means *this field decides the FK value*.
> **`mappedBy` names the Java field on the owning entity**, not a column.
> If you change only the inverse collection, **nothing is written** to the DB.

## 5. ManyToOne / OneToMany (lecture code)
```java
@Entity @Getter @Setter @NoArgsConstructor
public class Student {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    private String name;

    @ManyToOne                          // OWNING side
    @JoinColumn(name = "dept_id")       // FK column in students table
    private Department department;
}

@Entity @Getter @Setter @NoArgsConstructor
public class Department {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    private String name;

    @OneToMany(mappedBy = "department")  // INVERSE side; "department" = Student.department field
    private List<Student> students = new ArrayList<>();
}
```
Tables: `department(id, name)` and `student(id, name, dept_id → department.id)`.

Mandatory relation: `@ManyToOne(optional = false)` + `@JoinColumn(name = "dept_id", nullable = false)` (JPA rule + DB NOT NULL).

### Service: add a student to an existing department
```java
@Transactional
public void createStudent(Student student, Long deptId) {
    Department dept = departmentRepository.getDepartmentById(deptId);   // managed
    student.setDepartment(dept);            // owning side → sets dept_id
    dept.getStudents().add(student);        // inverse side → keep the object graph consistent
    studentRepository.save(student);        // INSERT INTO student(name, dept_id) VALUES (?, 5)
}
```
The existing department is **not** inserted again; only its id is used as the FK.

### Both new: persist order or cascade
```java
@Transactional
public void createStudent(Student student, String deptName) {
    Department dept = new Department(); dept.setName(deptName);
    student.setDepartment(dept);
    dept.getStudents().add(student);
    departmentRepository.save(dept);      // parent first
    studentRepository.save(student);
}
```
Without cascade, every new entity must be persisted explicitly. Otherwise you get `TransientPropertyValueException: object references an unsaved transient instance`.
With `@OneToMany(mappedBy="department", cascade = CascadeType.PERSIST)` (or `ALL`), persisting the department also persists its students.
> **Cascade controls lifecycle propagation. It does NOT decide ownership.**

### Changing the relationship = dirty checking
```java
student.setDepartment(hrDept);    // managed student → at flush: UPDATE student SET dept_id=20 WHERE id=1
```

### Helper methods (keep both sides in sync)
```java
// in Department
public void addStudent(Student s)    { students.add(s);    s.setDepartment(this); }
public void removeStudent(Student s) { students.remove(s); s.setDepartment(null); }
```
> ⚠️ The lecture's commented `Student.removeDepartment()` does `this.department = null;` **then** `this.department.remove(this)`, which is a guaranteed `NullPointerException` (and `Department` has no `remove`). Update the collection first, then null the reference, as above.

## 6. OneToOne
```java
@Entity public class Profile {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    private String bio;
    @OneToOne
    @JoinColumn(name = "user_id")        // OWNING side (has FK). Add unique = true to really enforce 1:1
    private User user;
}

@Entity public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    private String name;
    @OneToOne(mappedBy = "user")         // INVERSE
    private Profile profile;
}
```
FK alone doesn't guarantee one-to-one; you need **FK + UNIQUE**. (Recent Hibernate versions usually add a unique constraint for `@OneToOne` join columns in generated schemas, but don't rely on it. Declare `unique = true`.)

**Shared primary key** (`Profile.id` = `User.id`):
```java
@OneToOne @MapsId @JoinColumn(name = "id")
private User user;
```
> ⚠️ Naming an entity `User` maps to a table called `user`, which is a reserved word in PostgreSQL and some other DBs. Prefer `@Table(name = "users")`.

## 7. ManyToMany
```java
@Entity public class Student2 {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    private String name;

    @ManyToMany                                         // OWNING side
    @JoinTable(name = "student_course",
               joinColumns        = @JoinColumn(name = "student_id"),   // FK → this entity
               inverseJoinColumns = @JoinColumn(name = "course_id"))    // FK → other entity
    private List<Course> courses = new ArrayList<>();
}

@Entity public class Course {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    private String name;
    @ManyToMany(mappedBy = "courses")                   // INVERSE
    private List<Student2> students = new ArrayList<>();
}
```
Join table `student_course(student_id, course_id)`, ideally with a UNIQUE (student_id, course_id) constraint (`uniqueConstraints = @UniqueConstraint(columnNames = {"student_id","course_id"})`).
`student.getCourses().add(java)` → `INSERT INTO student_course VALUES (1, 101)`.
Helpers: `addCourse(c) { courses.add(c); c.getStudents().add(this); }`.

> **Prefer `Set` over `List` for `@ManyToMany`.** With a `List` (a Hibernate "bag"), removing one element often makes Hibernate **delete all join rows for that student and re-insert the rest**.
> If the link needs extra columns (enrolledAt, grade), model it as its own entity (`Enrollment`) with two `@ManyToOne`s instead.

## 8. Fetch types (defaults matter)
| Annotation | Default fetch |
|---|---|
| `@ManyToOne`, `@OneToOne` | **EAGER** |
| `@OneToMany`, `@ManyToMany`, `@ElementCollection` | **LAZY** |
Best practice: make everything LAZY (`@ManyToOne(fetch = FetchType.LAZY)`) and fetch what you need with `JOIN FETCH` or entity graphs. This avoids the **N+1 select problem**. Accessing a LAZY collection after the transaction ends gives `LazyInitializationException`.

## 9. Lombok with entities
Use `@Getter @Setter @NoArgsConstructor` (as the lecture does). **Avoid `@Data`, `@ToString`, `@EqualsAndHashCode` on entities with relationships**: bidirectional fields cause infinite recursion (`StackOverflowError`), unexpected lazy loading, and broken hash-based collections. Base `equals`/`hashCode` on a stable business key or id.
Also, returning bidirectional entities as JSON recurses forever (`Department → students → department → …`). Return DTOs, or use `@JsonManagedReference`/`@JsonBackReference`/`@JsonIgnore`.

## ⚠️ Issues in the lecture code
1. `getDepartmentById()` returns `null` for an unknown id, so `createStudent(student, deptId)` throws a `NullPointerException` → 500. Throw a `ResourceNotFoundException` instead.
2. `createDepartment(department, studentName)` saves the **student before the department** (child before parent, no cascade). Persist the parent first or add cascade.
3. `@ManyToMany` uses `List` (bag semantics); `Set` is preferred.
4. The `@OneToOne` join column has no explicit `unique = true`.
5. Entities are bound directly from `@RequestBody` (no DTOs), so clients could send nested objects or ids.

## 10. Quick reference
| Mapping | Owning side |
|---|---|
| OneToMany/ManyToOne | `@ManyToOne` + `@JoinColumn` |
| OneToOne | side with `@JoinColumn` |
| ManyToMany | side with `@JoinTable` |

---

## Interview Q&A
**Q1. What is the owning side of a relationship?**
The side whose mapping (`@JoinColumn`/`@JoinTable`) JPA uses to write the FK or join rows. The other side uses `mappedBy` and is ignored for writes.

**Q2. What does `mappedBy` refer to?**
The **field name** on the owning entity, not a column name.

**Q3. I added a student to `department.getStudents()` but `dept_id` stayed NULL. Why?**
You only changed the inverse side. Set `student.setDepartment(dept)` (use helper methods).

**Q4. How is ManyToMany stored?**
In a join table with two FKs (`@JoinTable`, `joinColumns`, `inverseJoinColumns`).

**Q5. Default fetch types? What is N+1?**
`*ToOne` are EAGER; `*ToMany` are LAZY. N+1 = 1 query for parents + N queries for each parent's association. Fix with `JOIN FETCH`, entity graphs, or batch fetching.

**Q6. What is cascade? Does it define ownership?**
Propagation of operations (PERSIST, MERGE, REMOVE…) to related entities. It's independent of ownership.

**Q7. How do you enforce one-to-one at the DB level?**
FK + UNIQUE constraint, or a shared primary key with `@MapsId`.

**Q8. Why avoid `@Data` on JPA entities?**
Generated `toString`/`equals`/`hashCode` traverse relationships, causing recursion, lazy loading, and unstable hashes.

**Q9. `List` vs `Set` for `@ManyToMany`?**
A `Set` lets Hibernate add or remove individual join rows. A `List` (bag) often deletes and re-inserts all rows.

**Q10. What is `orphanRemoval`?**
`@OneToMany(orphanRemoval = true)` deletes a child row when it's removed from the parent's collection.

## Quick revision
- `*ToOne` owns the FK via `@JoinColumn`; `mappedBy` = the inverse side, a field name.
- Bidirectional = one FK, two Java fields; sync both with helper methods.
- OneToOne = FK + UNIQUE (or `@MapsId`); ManyToMany = `@JoinTable` (use `Set`).
- Cascade ≠ ownership. Persist the parent first or cascade.
- Lazy everything + `JOIN FETCH`; no `@Data` on entities; return DTOs.
