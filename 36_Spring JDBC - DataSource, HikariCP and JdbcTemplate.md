# Spring JDBC: DataSource, HikariCP and JdbcTemplate

> **Source:** Lecture 27 (`Notes_Of_Aditya/Lecture 27/code/springJDBCDemo`)
> **Prerequisite:** [35_JDBC Fundamentals](35_JDBC%20Fundamentals.md) · **Next:** [37_Hibernate and JPA Entity Mapping](37_Hibernate%20and%20JPA%20Entity%20Mapping.md)

## Why this topic matters
`JdbcTemplate` keeps full SQL control without raw-JDBC boilerplate. `DataSource` and connection pooling (HikariCP) apply to **every** Spring data access stack, JPA included. Pool sizing and `queryForObject` behaviour are common interview topics.

---

## 1. What Spring JDBC removes
| You still write | Spring handles |
|---|---|
| SQL | Getting and releasing connections |
| Parameter values (in order) | Creating `PreparedStatement`, binding |
| Row → object mapping (`RowMapper`) | Iterating the `ResultSet` |
| Interpreting results | Closing resources |
| | Translating `SQLException` → `DataAccessException` |

This is the **Template Method pattern**: Spring owns the fixed workflow, you supply the variable parts.

## 2. `DataSource`: connection provider
```java
javax.sql.DataSource → Connection getConnection();
```
| `DriverManager` | `DataSource` |
|---|---|
| Static; repository knows URL/user/password | **Injectable** abstraction; config lives outside the repository |
| New physical connection each call | Implementation may be **pooled** |

> `DataSource` ≠ pool. A pool (e.g. `HikariDataSource`) **implements** `DataSource`. Your code depends on the interface (Dependency Inversion).

## 3. Connection pooling and HikariCP
Without a pool: every request does TCP connect, auth, session, query, close. That's slow, and many concurrent requests can overwhelm the DB.
With a pool:
```
borrow connection → run SQL → close() → RETURNED to pool (not destroyed)
all busy? → caller WAITS up to connection-timeout → then exception
```
The limit protects the database. **HikariCP** is Boot's default pool (comes with `spring-boot-starter-jdbc` / `-data-jpa`).
```properties
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000   # ms to wait for a free connection
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```
A bigger pool isn't automatically faster. Size it from concurrency, query time, DB capacity, number of app instances, and load tests.

## 4. Spring Boot setup
```xml
<dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-jdbc</artifactId></dependency>
<dependency><groupId>com.mysql</groupId><artifactId>mysql-connector-j</artifactId><scope>runtime</scope></dependency>
<dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-webmvc</artifactId></dependency>
```
```properties
spring.datasource.url=jdbc:mysql://localhost:3306/student_db
spring.datasource.username=root
spring.datasource.password=123456
```
Boot auto-config: detects spring-jdbc + driver → reads `spring.datasource.*` → creates a `HikariDataSource` bean → creates a `JdbcTemplate(dataSource)` bean. Define your own `DataSource` bean and Boot backs off.
Check it: inject `DataSource` and print `getClass()` → `class com.zaxxer.hikari.HikariDataSource`.

## 5. Repository with `JdbcTemplate` (lecture code)
```java
@Repository
public class StudentRepository {
    private final JdbcTemplate jdbcTemplate;
    private final RowMapper<Student> rowMapper = new BeanPropertyRowMapper<>(Student.class);
    // or: private final RowMapper<Student> rowMapper = new StudentRowMapper();

    public StudentRepository(JdbcTemplate jdbcTemplate) { this.jdbcTemplate = jdbcTemplate; }

    public void createStudent(Student s) {
        String sql = "INSERT INTO students(name, email, age) VALUES(?, ?, ?)";
        int rows = jdbcTemplate.update(sql, s.getName(), s.getEmail(), s.getAge());  // order = placeholder order
    }

    public void updateStudent(Student s, Long id) {
        jdbcTemplate.update("UPDATE students SET name=?, email=?, age=? WHERE id=?",
                s.getName(), s.getEmail(), s.getAge(), id);
    }

    public void deleteStudent(Long id) {
        jdbcTemplate.update("DELETE FROM students WHERE id = ?", id);
    }

    public Student getStudentById(Long id) {
        return jdbcTemplate.queryForObject("SELECT id, name, email, age FROM students WHERE id = ?", rowMapper, id);
    }

    public List<Student> getStudent() {
        return jdbcTemplate.query("SELECT id, name, email, age FROM students", rowMapper);
    }
}
```
| Method | Use | Returns |
|---|---|---|
| `update(sql, args...)` | INSERT / UPDATE / DELETE | affected rows (`int`) |
| `query(sql, rowMapper, args...)` | 0..n rows | `List<T>` (**empty list** if none) |
| `queryForObject(sql, rowMapper, args...)` | exactly 1 row | `T`. **0 rows → `EmptyResultDataAccessException`**, >1 → `IncorrectResultSizeDataAccessException` |
| `queryForObject(sql, Integer.class)` | scalar (count) | value |
| `batchUpdate(...)` | bulk DML | `int[]` |
Also: `NamedParameterJdbcTemplate` (`:name` params instead of `?`), and `JdbcClient` (fluent API, Spring 6.1+).

## 6. RowMapper
```java
public class StudentRowMapper implements RowMapper<Student> {       // one row → one object
    @Override
    public Student mapRow(ResultSet rs, int rowNum) throws SQLException {
        Student s = new Student();
        s.setId(rs.getLong("id"));
        s.setName(rs.getString("name"));
        s.setEmail(rs.getString("email"));
        s.setAge(rs.getInt("age"));
        return s;
    }
}
// lambda form:
RowMapper<Student> m = (rs, n) -> new Student(rs.getLong("id"), rs.getString("name"), ...);
```
Stateless, so a single instance can be reused.

**`BeanPropertyRowMapper<>(Student.class)`**: maps columns to JavaBean properties automatically (`first_name` → `firstName`). It needs a no-arg constructor and setters. It's convenient but less explicit, uses reflection, and silently leaves unmatched columns null. A custom `RowMapper` is better for joins or complex mappings.

## 7. Not-found handling
```java
public Optional<Student> findById(Long id) {
    try {
        return Optional.of(jdbcTemplate.queryForObject(sql, rowMapper, id));
    } catch (EmptyResultDataAccessException e) {     // ONLY this one means "not found"
        return Optional.empty();
    }
}
// or: return jdbcTemplate.query(sql, rowMapper, id).stream().findFirst();
```
Don't catch all `DataAccessException`s and return empty, because "DB down" isn't "not found".

## 8. Exception translation
`SQLException` is **checked**, broad, and vendor-specific (`getSQLState()`, `getErrorCode()`).
Spring translates it into the **unchecked**, vendor-neutral `DataAccessException` hierarchy:
| Exception | Cause |
|---|---|
| `DuplicateKeyException` | Unique/PK violation (e.g. duplicate email) |
| `DataIntegrityViolationException` | FK / NOT NULL / check constraint |
| `BadSqlGrammarException` | Invalid SQL / unknown column |
| `CannotGetJdbcConnectionException` | DB down, bad credentials, pool timeout |
| `EmptyResultDataAccessException` | `queryForObject` found 0 rows |
| `IncorrectResultSizeDataAccessException` | Wrong row count |
```java
try { repo.save(s); } catch (DuplicateKeyException e) { throw new EmailAlreadyExistsException(s.getEmail()); }
```
Portable across MySQL/PostgreSQL, and no `throws SQLException` everywhere. `@Repository` classes get the same translation for other technologies.

## ⚠️ Issues in the lecture code
1. `GET /api/student/{id}` with a missing id: `queryForObject` throws `EmptyResultDataAccessException`, which becomes a **500**, not a 404. Map it to `Optional` or a custom exception.
2. `PUT /api/student` takes the id from the request body (no `{id}` in the path). That isn't RESTful, and a missing id updates nothing silently (`update` returns 0, and the API still answers "DONE").
3. Create/update/delete return "DONE" even when 0 rows were affected. Return the count or 404.
4. Raw `Student` used as the API model (no DTO/validation).

## 9. Raw JDBC vs Spring JDBC
| | Raw JDBC | Spring JDBC |
|---|---|---|
| SQL control | Full | Full |
| Connections | `DriverManager` by hand | `DataSource` + pool |
| Params / iteration / closing | Manual | Automatic |
| Mapping | Manual `ResultSet` code | `RowMapper` |
| Exceptions | Checked `SQLException` | Unchecked `DataAccessException` |

---

## Interview Q&A
**Q1. What is a DataSource? Is it a connection pool?**
An interface for obtaining connections. Not necessarily a pool: pools like HikariCP *implement* it.

**Q2. Why connection pooling?**
Creating connections is expensive (network + auth). A pool reuses them and caps concurrency to protect the DB.

**Q3. What does `connection.close()` do with a pool?**
Returns the connection to the pool.

**Q4. Which pool does Spring Boot use by default?**
HikariCP.

**Q5. `query` vs `queryForObject`?**
`query` returns a list (possibly empty). `queryForObject` expects exactly one row and throws `EmptyResultDataAccessException` if none.

**Q6. What is a RowMapper? `BeanPropertyRowMapper`?**
RowMapper maps one row to one object. BeanPropertyRowMapper does it reflectively by matching column and property names.

**Q7. How does Spring handle SQLExceptions?**
Translates them into the unchecked `DataAccessException` hierarchy (`DuplicateKeyException`, etc.).

**Q8. JdbcTemplate vs JPA: when would you choose JdbcTemplate?**
For complex, hand-tuned SQL, reporting, bulk operations, or when you want no ORM magic. JPA suits rich domain models with standard CRUD.

**Q9. What happens when all pool connections are busy?**
The caller waits up to `connection-timeout`, then gets an exception.

## Quick revision
- DataSource = connection provider; HikariCP = Boot's default pool implementation.
- `update` = DML, `query` = list, `queryForObject` = exactly one (throws on 0).
- RowMapper maps one row; BeanPropertyRowMapper is the reflective shortcut.
- `SQLException` → `DataAccessException` (unchecked, portable).
- Distinguish "not found" from "DB error".
