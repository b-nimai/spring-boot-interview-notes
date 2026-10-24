# Spring Data JPA: Entity, Repository and Database Config

> **Source:** Lecture 12 (`Notes_Of_Aditya/Lecture 12/code/crudSpringBootDemo`), Lecture 11 (MySQL setup)
> **Related:** [18_Layered Architecture](18_Layered%20Architecture%20and%20REST%20CRUD%20API.md) · [20_Soft Delete](20_Soft%20Delete%20and%20Derived%20Query%20Methods.md)

## Why this topic matters
Almost every Spring Boot backend talks to a relational DB through JPA. You must be able to explain ORM, `@Entity`, `JpaRepository`, `ddl-auto`, and why `findById` returns an `Optional`.

---

## 1. The stack
```
Your code → Spring Data JPA → JPA (spec) → Hibernate (implementation) → JDBC → MySQL driver → MySQL
```
- **JPA** = specification (annotations and interfaces: `@Entity`, `EntityManager`).
- **Hibernate** = the ORM implementation that generates SQL.
- **Spring Data JPA** = generates repository implementations, so you write only interfaces.
- **ORM (Object-Relational Mapping):** class ↔ table, object ↔ row, field ↔ column.

## 2. MySQL setup (macOS)
```bash
brew install mysql
brew services start mysql
mysql_secure_installation          # set root password
mysql -u root -p
```
```sql
CREATE DATABASE student_crud_db;   -- create the DB once; Hibernate creates the TABLES
```
Default port: **3306**. GUI tools: MySQL Workbench, Sequel Ace, DBeaver.

## 3. `application.properties`
```properties
spring.datasource.url=jdbc:mysql://localhost:3306/student_crud_db
spring.datasource.username=root
spring.datasource.password=123456

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
# spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver   (usually auto-detected)
```
URL breakdown: `jdbc` (protocol) : `mysql` (DB type) :// `localhost` : `3306` / `student_crud_db`.

### `ddl-auto` values
| Value | Behaviour | Use |
|---|---|---|
| `none` | Hibernate doesn't touch the schema | Production |
| `validate` | Checks that the schema matches the entities; fails if not | Production (with migrations) |
| `update` | Creates or alters tables to match entities; **never drops columns** | Local dev |
| `create` | Drops and recreates tables at every startup (**data lost**) | Tests/demos |
| `create-drop` | Create on start, drop on shutdown | Tests |

> Production uses **Flyway / Liquibase** migration scripts, not `update`.

## 4. Entity
```java
@Entity                                  // JPA-managed class → table
@Table(name = "students")                // optional; default table name comes from the class name
public class Student {

    @Id                                              // primary key
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // MySQL AUTO_INCREMENT
    private Long id;

    private String name;
    private int age;
    private String email;
    private int rollNo;                  // column roll_no (camelCase → snake_case by Spring's naming strategy)
    private String subject;

    // JPA needs a no-arg constructor (implicit here) + getters/setters
}
```
| `GenerationType` | Meaning |
|---|---|
| `IDENTITY` | DB auto-increment column (MySQL) |
| `SEQUENCE` | DB sequence (PostgreSQL, Oracle) |
| `TABLE` | Separate key table (slow, rare) |
| `AUTO` | Let Hibernate choose |

> ⚠️ **Lecture 12's entity has `@Id` but no `@GeneratedValue`.** With a `Long id` the client must send an `id`; if it's `null`, `save()` fails with *"Identifier of entity must be manually assigned"*. Lecture 13 adds `@GeneratedValue(strategy = GenerationType.IDENTITY)`, which is the fix.

Other useful mappings: `@Column(name="roll_no", nullable=false, unique=true, length=50)`, `@Transient` (not persisted), `@Enumerated(EnumType.STRING)`.

## 5. Repository: an interface, not a class
```java
public interface StudentRepository extends JpaRepository<Student, Long> {   // <Entity, ID type>
}
```
- **No implementation class.** Spring Data creates a proxy implementation at runtime.
- **No `@Repository` needed.** Interfaces extending `JpaRepository` are detected automatically. They still get exception translation.

### Hierarchy
```
Repository → CrudRepository → ListCrudRepository → PagingAndSortingRepository → JpaRepository
```
`JpaRepository` adds `flush()`, `saveAndFlush()`, batch deletes, and `List` return types.

### Methods used in the lecture
| Method | Returns | Notes |
|---|---|---|
| `save(entity)` | saved entity | **INSERT if id is null / new, UPDATE otherwise** (merge) |
| `findById(id)` | `Optional<T>` | Forces you to handle "not found" |
| `findAll()` | `List<T>` | All rows |
| `existsById(id)` | `boolean` | Cheap existence check |
| `deleteById(id)` / `delete(entity)` | void | Hard delete |
| `count()` | long | |

## 6. Service using the repository (Lecture 12)
```java
@Service
public class StudentService {
    private StudentRepository studentRepository;

    public Student createStudent(Student req) { return studentRepository.save(req); }

    public Student getStudent(Long id) {
        Optional<Student> resp = studentRepository.findById(id);
        return resp.isPresent() ? resp.get() : null;      // Lecture 17 replaces null with an exception
    }

    public Student updateStudent(Long id, Student req) {
        Optional<Student> existing = studentRepository.findById(id);
        if (existing.isEmpty()) return null;
        Student s = existing.get();                         // 1. fetch
        s.setName(req.getName());                           // 2. modify allowed fields
        s.setAge(req.getAge()); /* ... */
        return studentRepository.save(s);                   // 3. save → UPDATE
    }

    public Boolean deleteStudent(Long id) {
        if (!studentRepository.existsById(id)) return false;
        studentRepository.deleteById(id);
        return true;
    }
}
```
**Update pattern: fetch → modify → save.** Never build a brand-new entity for an update, or you'll overwrite columns you didn't set.

`show-sql` output for `save()` on a new entity:
```sql
insert into student (age, email, name, roll_no, subject) values (?, ?, ?, ?, ?)
```

## 7. Why `Optional`?
The row may not exist. `Optional` makes that explicit:
```java
studentRepository.findById(id)
    .orElseThrow(() -> new ResourceNotFoundException("Student " + id + " not found"));
```
Calling `.get()` on an empty Optional throws `NoSuchElementException`, which becomes a 500 error.

---

## Common pitfalls
- DB not created → `Unknown database 'student_crud_db'`. Hibernate creates tables, not databases.
- Committing real DB passwords in `application.properties` → use env vars (`${DB_PASSWORD}`) or profiles (Lecture 18).
- `ddl-auto=create` in a shared DB wipes data on every restart.
- Missing no-arg constructor on an entity → `InstantiationException`.

## Interview Q&A
**Q1. JPA vs Hibernate vs Spring Data JPA?**
JPA is the spec. Hibernate implements it. Spring Data JPA sits on top and generates repository implementations plus derived queries.

**Q2. How does `save()` decide between INSERT and UPDATE?**
If the entity is new (id `null`, or `isNew()` true), it calls `persist` → INSERT. Otherwise `merge` → SELECT + UPDATE.

**Q3. Why is `StudentRepository` an interface without an implementation?**
Spring Data creates a runtime proxy (backed by `SimpleJpaRepository`) that implements the interface.

**Q4. What does `ddl-auto=update` do, and why not use it in production?**
It alters the schema to match entities. It can't drop or rename safely, and changes are untracked. Production uses versioned migrations (Flyway/Liquibase) with `validate` or `none`.

**Q5. `IDENTITY` vs `SEQUENCE`?**
IDENTITY relies on DB auto-increment (no JDBC batch inserts). SEQUENCE uses a DB sequence and supports batching and pre-allocation.

**Q6. Why does `findById` return `Optional`?**
To force callers to handle absence explicitly instead of risking a `NullPointerException`.

**Q7. `CrudRepository` vs `JpaRepository`?**
`JpaRepository` extends `CrudRepository` and `PagingAndSortingRepository` and adds JPA-specific methods (flush, batch delete, `List` returns).

**Q8. `deleteById` vs `delete(entity)`?**
`deleteById` loads the entity first, then deletes it (in recent versions it silently ignores a missing id). `delete(entity)` deletes an entity you already have.

## Quick revision
- `@Entity` + `@Id` + `@GeneratedValue(IDENTITY)` = table with auto-increment PK.
- `interface XRepository extends JpaRepository<Entity, IdType>` with no implementation.
- `save` = insert or update; `findById` = `Optional`; update = fetch → modify → save.
- `ddl-auto=update` for local dev only; migrations in production.
