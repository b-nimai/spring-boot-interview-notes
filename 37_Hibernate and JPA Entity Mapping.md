# Hibernate and JPA Entity Mapping

> **Source:** Lecture 28 (`Notes_Of_Aditya/Lecture 28/code/hibernateDemo`)
> **Related:** [19_Spring Data JPA](19_Spring%20Data%20JPA%20-%20Entity%2C%20Repository%20and%20Database%20Config.md) · **Next:** [38_Hibernate Internals](38_Hibernate%20Internals%20-%20Persistence%20Context%2C%20Dirty%20Checking%20and%20Flush.md) · [39_JPA Relationships](39_JPA%20Relationships.md)

## Why this topic matters
This lecture uses **`EntityManager` directly** (no `JpaRepository`), which shows what Spring Data hides. Interviewers ask about ORM, JPA vs Hibernate, `@Column` options, enums, `@Embeddable`, `@ElementCollection`, and converters.

---

## 1. Object-relational impedance mismatch
| Java | Relational DB |
|---|---|
| Classes, objects, references | Tables, rows, foreign keys |
| Inheritance, encapsulation | No inheritance |
| Collections | Join/child tables |
| Object identity (`==`) | Primary key |

With JdbcTemplate you still hand-write the SQL, the parameter order, and the RowMapper for **every** entity. **ORM** declares the mapping **once** and reuses it for every insert, select, update, and delete.

ORM responsibilities: mapping metadata, SQL generation, parameter binding, result mapping, relationship management, caching (first-level / optional second-level), and transaction integration.

## 2. JPA vs Hibernate vs Spring Boot
```
Application → Jakarta Persistence API (JPA: spec) → Hibernate (provider/impl) → JDBC → Driver → DB
```
- **JPA / Jakarta Persistence** = the specification: `@Entity`, `@Id`, `EntityManager`, `EntityManagerFactory`… (package `jakarta.persistence`).
- **Hibernate** = the most popular implementation (others: EclipseLink, OpenJPA). Like `List` (spec) vs `ArrayList` (impl).
- **Spring Boot** `spring-boot-starter-data-jpa` brings Hibernate + Spring ORM + Spring Data JPA and auto-configures `DataSource` → `EntityManagerFactory` → `JpaTransactionManager`.
Code against `jakarta.persistence.*` to stay provider-neutral.

## 3. Configuration
```properties
spring.datasource.url=jdbc:mysql://localhost:3306/student_db
spring.datasource.username=root
spring.datasource.password=123456
spring.jpa.hibernate.ddl-auto=update          # none | validate | update | create | create-drop
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```
(See [19](19_Spring%20Data%20JPA%20-%20Entity%2C%20Repository%20and%20Database%20Config.md) for `ddl-auto` details. Use Flyway/Liquibase in production.)

## 4. CRUD with `EntityManager` (lecture code)
```java
@Repository
public class StudentRepository {
    @PersistenceContext                       // inject a (proxy to the) transaction-bound EntityManager
    private EntityManager entityManager;

    public void save(Student s)       { entityManager.persist(s); }                 // INSERT (at flush)
    public Student findById(Long id)  { return entityManager.find(Student.class, id); } // SELECT or cache; null if absent
    public void remove(Student s)     { entityManager.remove(s); }                  // DELETE (at flush), entity must be managed
}

@Service
public class StudentService {
    @Transactional
    public void updateStudent(Student req, Long id) {
        Student s = studentRepository.findById(id);            // managed entity
        if (s == null) throw new RuntimeException("Student not found");
        s.setName(req.getName());                              // NO save()/update() call needed:
        s.setAge(req.getAge());                                // dirty checking generates the UPDATE at commit
        s.setEmail(req.getEmail());
    }

    @Transactional
    public void deleteStudent(Long id) {
        Student s = studentRepository.findById(id);
        if (s == null) throw new RuntimeException("Student not found");
        studentRepository.remove(s);
    }
}
```
There's no `entityManager.update()` in JPA. Changes to a **managed** entity inside a transaction are flushed automatically. See [38](38_Hibernate%20Internals%20-%20Persistence%20Context%2C%20Dirty%20Checking%20and%20Flush.md).
> Note: the lecture imports `jakarta.transaction.Transactional` (JTA). Spring supports it, but **`org.springframework.transaction.annotation.Transactional`** is the usual choice: it has `readOnly`, `propagation`, `isolation`, and `rollbackFor`.

## 5. Column mapping
```java
@Entity
@Table(name = "students")
public class Student {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "student_name", nullable = false, length = 100)
    private String name;

    @Column(unique = true, nullable = false, length = 150, insertable = true, updatable = true)
    private String email;

    private int age;

    protected Student() {}            // JPA needs a no-arg constructor (can be protected)
}
```
| `@Column` attribute | Effect |
|---|---|
| `name` | Column name |
| `nullable`, `unique`, `length` | **Schema generation** (DDL) constraints |
| `insertable` / `updatable` | Include in INSERT / UPDATE statements (e.g. `updatable=false` for `createdAt`) |
| `precision`, `scale` | Decimals: `@Column(precision=5, scale=2) BigDecimal percentage` → 98.75 |

### More mappings (lecture's commented code)
```java
private LocalDate dateOfBirth;              // DATE
private LocalDateTime createdAt;            // TIMESTAMP (prefer java.time over java.util.Date)

@Enumerated(EnumType.STRING)                // stores "ACTIVE" — default ORDINAL stores 0,1,2 (breaks if enum order changes!)
private StudentStatus status;               // enum ACTIVE, INACTIVE, SUSPENDED

@Lob private String profileDescription;     // large text (CLOB/LONGTEXT); @Lob byte[] for binary
@Transient private String displayName;      // NOT persisted

@Convert(converter = BooleanToStringConverter.class)
private Boolean isMonitor;                  // stored as "Yes"/"No"
```
### AttributeConverter
```java
public class BooleanToStringConverter implements AttributeConverter<Boolean, String> {
    public String convertToDatabaseColumn(Boolean b)  { return b == null ? null : (b ? "Yes" : "No"); }
    public Boolean convertToEntityAttribute(String s) { return s == null ? null : "Yes".equals(s); }
}
// @Converter(autoApply = true) on the class would apply it to every Boolean automatically
```

## 6. Value types: `@Embeddable` / `@Embedded`
**Entity** = independent identity (Student, Department). **Value type** = no own id; lifecycle owned by an entity (Address).
```java
@Embeddable
public class Address {
    private String houseNo, street, city, state, pincode;
    public Address() {}
}

@Entity
public class Student {
    @Embedded
    private Address address;      // columns house_no, street, city... live in the STUDENTS table
}
```
Two addresses of the same type means column clash, so rename with `@AttributeOverrides`:
```java
@Embedded
@AttributeOverrides({
    @AttributeOverride(name = "city", column = @Column(name = "current_city")),
    @AttributeOverride(name = "state", column = @Column(name = "current_state"))
    // … one per field
})
private Address currentAddress;
// permanentAddress → permanent_city, permanent_state …
```

## 7. Collections of values: `@ElementCollection`
```java
@ElementCollection
@CollectionTable(name = "student_address", joinColumns = @JoinColumn(name = "student_id"))
private Set<Address> addresses;               // lecture: separate table student_address(student_id, house_no, ...)

@ElementCollection
@CollectionTable(name = "student_skills", joinColumns = @JoinColumn(name = "student_id"))
@Column(name = "skill")
private Set<String> skills = new HashSet<>(); // student_skills(student_id, skill)
```
The values have no identity (you can't `find(Skill.class, id)`) and are owned by the Student (deleted with it). They're lazily loaded by default.
> ⚠️ In the lecture, `Address` has **no `equals()`/`hashCode()`** but is stored in a `Set`. Embeddables in Sets should implement value-based `equals`/`hashCode`, otherwise duplicate detection and removal don't work as expected. Also initialize collections (`= new HashSet<>()`) to avoid NPEs.
> Updating an element collection typically makes Hibernate **delete all rows and re-insert** them. For large or frequently-changing collections, use a separate `@Entity` with `@OneToMany`.

| Use `@Embedded`/`@ElementCollection` when | Use an `@Entity` + relationship when |
|---|---|
| No independent identity, never shared, never queried alone | Shared (Department), queried or updated independently, own lifecycle |

---

## Interview Q&A
**Q1. What is ORM? What problem does it solve?**
Mapping objects to relational tables declaratively, which removes hand-written SQL and mapping and bridges the object-relational impedance mismatch.

**Q2. JPA vs Hibernate?**
JPA is the specification (interfaces and annotations). Hibernate is an implementation (provider).

**Q3. `persist` vs `find` vs `remove`?**
`persist` makes a new entity managed (INSERT at flush). `find` loads by PK (returns null if absent). `remove` schedules deletion of a managed entity.

**Q4. Why is there no `update()` method in JPA?**
Managed entities are dirty-checked; modifications inside a transaction are flushed automatically.

**Q5. `EnumType.ORDINAL` vs `STRING`?**
ORDINAL stores the position (fragile if constants are reordered or inserted). STRING stores the name (safe, readable). Prefer STRING.

**Q6. `@Embeddable` vs `@Entity`?**
An embeddable has no ID or lifecycle of its own and is stored in its owner's table. An entity has its own identity and table.

**Q7. `@ElementCollection` vs `@OneToMany`?**
ElementCollection holds values (basic or embeddable) in a collection table, owned entirely by the parent. OneToMany links independent entities.

**Q8. What does `@Transient` do?** What about `insertable/updatable=false`?
`@Transient` excludes a field from persistence. `insertable/updatable=false` keeps the mapping but leaves the column out of INSERT/UPDATE.

**Q9. What is an AttributeConverter?**
Custom conversion between an entity attribute type and a DB column type (e.g. Boolean ↔ "Yes"/"No", encryption).

## Quick revision
- ORM = declare mapping once; JPA = spec, Hibernate = impl.
- EntityManager: persist / find / remove; updates via dirty checking in `@Transactional`.
- `@Column` (name, nullable, unique, length, updatable), `@Enumerated(STRING)`, `@Lob`, `@Transient`, `@Convert`.
- `@Embeddable` + `@Embedded` (+ `@AttributeOverrides`) = same table; `@ElementCollection` = separate collection table.
