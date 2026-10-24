# Soft Delete and Derived Query Methods

> **Source:** Lecture 13 (`Notes_Of_Aditya/Lecture 13/code/crudSpringBootDemo`)
> **Related:** [19_Spring Data JPA](19_Spring%20Data%20JPA%20-%20Entity%2C%20Repository%20and%20Database%20Config.md)

## Why this topic matters
Real systems rarely delete rows permanently (audit, undo, legal reasons). Soft delete plus Spring Data's **method-name queries** come up often in interviews and in real code.

---

## 1. Hard delete vs soft delete
| | Hard delete | Soft delete |
|---|---|---|
| What happens | Row removed (`DELETE FROM`) | Row stays; a flag is set (`deleted = true`) |
| HTTP method | `DELETE` | `PATCH` (it's a partial update of one field) |
| Recoverable | No | Yes |
| Lecture endpoint | `DELETE /api/students/delete?id=1` | `PATCH /api/students/delete-soft?id=1` |

## 2. Entity change
```java
@Entity
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)   // added in this lecture
    private Long id;
    // ... name, age, email, rollNo, subject
    private Boolean deleted;                               // false = active, true = soft-deleted
}
```
The service sets `deleted = false` on create.

## 3. Derived query methods
Spring Data **parses the method name** and generates the JPQL/SQL. No implementation and no `@Query` needed.
```java
public interface StudentRepository extends JpaRepository<Student, Long> {

    Optional<Student> findByIdAndDeletedIsFalse(Long id);
    // SELECT * FROM student WHERE id = ? AND deleted = false

    List<Student> findByDeletedIsFalse();
    // SELECT * FROM student WHERE deleted = false

    // findBy + FieldName + Condition
}
```
### How the name is parsed
```
findBy | Id | And | Deleted | IsFalse
  ↑      ↑    ↑       ↑         ↑
prefix field  op    field   condition
```
> The field name must match the **entity property** exactly. The lecture code uses a field `deleted`, so the method is `findByDeletedIsFalse`. The PDF uses a field `isDeleted`, so there it's `findByIsDeletedFalse`. A wrong name fails at **startup** with `No property 'xyz' found for type 'Student'`.

### Cheat sheet of keywords
| Keyword | Example | SQL |
|---|---|---|
| `And` / `Or` | `findByNameAndAge` | `name=? AND age=?` |
| `Is`, `Equals` | `findByEmailIs` | `email=?` |
| `True` / `False` | `findByDeletedFalse` | `deleted=false` |
| `IsNull` / `IsNotNull` | `findBySubjectIsNull` | `subject IS NULL` |
| `GreaterThan`, `LessThan`, `Between` | `findByAgeBetween(18, 25)` | `age BETWEEN ? AND ?` |
| `Like`, `Containing`, `StartingWith` | `findByNameContaining("ad")` | `name LIKE %ad%` |
| `In` | `findByRollNoIn(List)` | `roll_no IN (...)` |
| `OrderBy` | `findByDeletedFalseOrderByNameAsc` | `ORDER BY name` |
| `IgnoreCase` | `findByEmailIgnoreCase` | `UPPER(email)=UPPER(?)` |
| Other prefixes | `existsByEmail`, `countByDeletedFalse`, `deleteBy…`, `findFirstBy…`, `findTop3By…` | |

For complex queries use `@Query("SELECT s FROM Student s WHERE ...")` (JPQL) or `nativeQuery = true`.

## 4. Service (lecture code)
```java
public List<Student> getAllStudent() {
    return studentRepository.findByDeletedIsFalse();          // hide soft-deleted
}

public Student updateStudent(Long id, Student req) {
    Optional<Student> existing = studentRepository.findByIdAndDeletedIsFalse(id);
    if (existing.isEmpty()) return null;                       // can't update a deleted student
    ...
}

public Boolean deleteStudentSoftly(Long id) {
    Optional<Student> existing = studentRepository.findByIdAndDeletedIsFalse(id);
    if (existing.isEmpty()) return false;
    Student s = existing.get();
    s.setDeleted(true);                                        // flag, not DELETE
    studentRepository.save(s);                                 // → UPDATE student SET deleted=true ...
    return true;
}
```
Controller:
```java
@PatchMapping("/delete-soft")
public ResponseEntity<String> deleteStudentSoftly(@RequestParam Long id) {
    if (!studentService.deleteStudentSoftly(id)) return ResponseEntity.notFound().build();
    return ResponseEntity.ok("Record deleted");
}
```

## 5. ⚠️ Gotchas found in the lecture code
1. **`getStudent(id)` still uses `findById`**, so `GET /get?id=1` returns a soft-deleted student. Use `findByIdAndDeletedIsFalse(id)` for every read path.
2. **Old rows have `deleted = NULL`.** Rows inserted in Lecture 12 existed before the column. `ddl-auto=update` adds the column as NULL, and `WHERE deleted = false` **does not match NULL**, so those students disappear from `getAll`. Fixes:
   ```sql
   UPDATE student SET deleted = false WHERE deleted IS NULL;
   ```
   and/or a default: `private Boolean deleted = false;` with `@Column(nullable = false)`.
3. Soft delete is easy to forget in a new query. Hibernate offers `@SoftDelete` (Hibernate 6.4+) or `@SQLRestriction("deleted = false")` to apply the filter automatically.
4. Unique constraints (e.g. email) still apply to soft-deleted rows. Re-registering a deleted email will conflict.

---

## Interview Q&A
**Q1. What is soft delete and why use it?**
Marking a row as deleted (flag or `deleted_at` timestamp) instead of removing it. It gives you audit history, undo, referential safety, and helps with compliance.

**Q2. Which HTTP method for soft delete?**
`PATCH` is semantically accurate (partial update). Many APIs still expose it as `DELETE` and implement it as a soft delete. Either is defensible if documented.

**Q3. How do derived query methods work?**
At startup Spring Data parses the method name (`findBy` + properties + keywords), builds a JPQL query, and validates it against the entity metamodel. Invalid names fail fast.

**Q4. When would you use `@Query` instead?**
For joins, aggregates, projections, or when the method name gets unreadably long.

**Q5. How do you make soft delete automatic across all queries?**
Hibernate `@SoftDelete`, `@SQLRestriction` (formerly `@Where`), or filters. Otherwise every repository method must include the condition.

**Q6. Why might `findByDeletedFalse()` miss rows?**
Rows whose flag is `NULL`. SQL three-valued logic means `NULL = false` is not true.

## Quick revision
- Soft delete = flag + `save()`; hard delete = `deleteById`.
- `findBy<Field><Keyword>`: names must match entity fields exactly.
- Filter **every** read and update by `deleted = false`.
- Watch out for NULL flags on old rows.
