# JDBC Fundamentals

> **Source:** Lecture 26 (`Notes_Of_Aditya/Lecture 26/code/jdbcDemo`, plain Maven + `mysql-connector-j`)
> **Next:** [36_Spring JDBC](36_Spring%20JDBC%20-%20DataSource%2C%20HikariCP%20and%20JdbcTemplate.md)

## Why this topic matters
JPA, Hibernate, Spring Data, and JdbcTemplate all sit on top of JDBC. Interviewers ask about `Statement` vs `PreparedStatement`, SQL injection, `executeQuery` vs `executeUpdate`, ResultSet cursors, and resource closing.

---

## 1. How Java talks to a database
Java (JVM) and MySQL are **separate processes**, often on separate machines. `String sql = "SELECT …"` is just a string in JVM memory until something sends it.
```
Java code → JDBC API (java.sql interfaces) → JDBC Driver (MySQL Connector/J) → network protocol → MySQL Server
```
- **JDBC API** (`java.sql`): `Connection`, `Statement`, `PreparedStatement`, `ResultSet`, `SQLException`, `DriverManager`. A **standard contract**.
- **JDBC Driver**: the vendor's implementation that speaks that database's wire protocol (MySQL Connector/J, PostgreSQL JDBC, Oracle JDBC…).
- You code against interfaces, and switching DB means mostly changing the URL and driver (SQL dialects can still differ).

What happens on a query: JDBC call → driver encodes the request → server parses SQL, plans, executes → returns rows, update count, or error → driver decodes → `ResultSet` / `int`.

## 2. Setup
```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>9.6.0</version>          <!-- lecture; notes suggest <scope>runtime</scope> -->
</dependency>
```
```sql
CREATE DATABASE student_db;
USE student_db;
CREATE TABLE students (
  id    BIGINT PRIMARY KEY AUTO_INCREMENT,
  name  VARCHAR(100) NOT NULL,
  email VARCHAR(150) NOT NULL UNIQUE,
  age   INT
);
```
MySQL Server (the database), DBeaver/Workbench (a GUI **client**), and Connector/J (the Java driver) are three different things. Installing DBeaver doesn't start MySQL.

## 3. Connection
```java
String url = "jdbc:mysql://localhost:3306/student_db";   // jdbc:<db>://<host>:<port>/<database>
try (Connection con = DriverManager.getConnection(url, "root", "123456")) {
    System.out.println("Connected");
} catch (SQLException e) {
    System.out.println(e.getMessage() + " | SQLState=" + e.getSQLState() + " | code=" + e.getErrorCode());
}
```
- `Connection` = an active session (network + authentication). **Expensive to create**, which is why connection pools exist.
- `Class.forName("com.mysql.cj.jdbc.Driver")` is **not needed** since JDBC 4.0 (drivers auto-register via ServiceLoader).
- `DriverManager` doesn't read `application.properties`; that's a Spring thing.

## 4. Executing SQL
| Method | For | Returns |
|---|---|---|
| `executeQuery(sql)` | `SELECT` | `ResultSet` |
| `executeUpdate(sql)` | `INSERT` / `UPDATE` / `DELETE` (also DDL → 0) | `int` affected rows |
| `execute(sql)` | Unknown/mixed (stored procs, generic tools) | `boolean`: **true = first result is a ResultSet**, false = update count. *Not* success/failure |

## 5. ResultSet: a cursor, not a List
```
cursor starts BEFORE the first row → next() moves forward, returns false at the end
```
```java
while (rs.next()) {                                  // many rows
    long id = rs.getLong("id");                      // by label (clearer) or by index getLong(1) — 1-based!
    String name = rs.getString("name");
    Integer age = rs.getObject("age", Integer.class); // getInt() returns 0 for SQL NULL; use getObject or wasNull()
}
```
Prefer an explicit column list over `SELECT *` (predictable mapping, less data, safer when the schema changes).

## 6. `PreparedStatement` and SQL injection
❌ String concatenation:
```java
String sql = "SELECT * FROM students WHERE email = '" + email + "'";
// email = "x' OR '1'='1"  →  returns every row (SQL injection)
```
✅ Parameterized:
```java
String sql = "SELECT id, name, email, age FROM students WHERE email = ?";
PreparedStatement ps = con.prepareStatement(sql);
ps.setString(1, email);                 // parameter indexes start at 1
```
Benefits: **SQL injection protection**, correct quoting (`O'Brien`), proper NULL and date handling, and statement reuse or pre-compilation.
For NULLs: `ps.setNull(3, Types.INTEGER)` or `ps.setObject(3, age)`.
⚠️ Placeholders bind **values only**. They can't be table or column names or `ORDER BY` direction. Validate those against an allow-list.

## 7. CRUD (lecture code)
```java
public void createStudent(Student s) {
    String sql = """
                 INSERT INTO students(name, email, age)
                 VALUES(?, ?, ?)
                 """;
    try (Connection con = DriverManager.getConnection(url, username, password);
         PreparedStatement ps = con.prepareStatement(sql)) {
        ps.setString(1, s.getName());
        ps.setString(2, s.getEmail());
        ps.setInt(3, s.getAge());
        int rows = ps.executeUpdate();
        System.out.println(rows == 1 ? "Create Student successful" : "Create Student failed");
    } catch (SQLException e) {
        System.out.println("Database connection failed");
        e.printStackTrace();
    }
}

public void getStudentById(Long id) {
    String sql = "SELECT id, name, email, age FROM students WHERE id = ?";
    try (Connection con = DriverManager.getConnection(url, username, password);
         PreparedStatement ps = con.prepareStatement(sql)) {
        ps.setLong(1, id);
        try (ResultSet rs = ps.executeQuery()) {
            if (rs.next()) System.out.println(mapRow(rs));
        }
    } catch (SQLException e) { e.printStackTrace(); }
}

private Student mapRow(ResultSet rs) throws SQLException {    // reusable row → object mapping
    Student s = new Student();
    s.setId(rs.getLong("id"));
    s.setName(rs.getString("name"));
    s.setEmail(rs.getString("email"));
    s.setAge(rs.getInt("age"));
    return s;
}
```
Update: `UPDATE students SET name=?, email=?, age=? WHERE id=?` → `setLong(4, id)`.
Delete: `DELETE FROM students WHERE id=?` → `executeUpdate()` returns 1 (deleted) or 0 (not found).

### Getting the generated id
```java
PreparedStatement ps = con.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS);
ps.executeUpdate();
try (ResultSet keys = ps.getGeneratedKeys()) {
    if (keys.next()) {
        long id = keys.getLong(1);
    }
}
```

### Better repository design (from the notes)
Return values (`Optional<Student>`, `List<Student>`, `int`) and **rethrow** failures:
```java
public Optional<Student> findById(Long id) { ... return rs.next() ? Optional.of(mapRow(rs)) : Optional.empty(); }
catch (SQLException e) { throw new RuntimeException("Failed to fetch student " + id, e); }
```
"Not found" (`Optional.empty()`) is not the same as "DB down" (an exception). Don't merge them.

## 8. Closing resources
```
Connection → creates → PreparedStatement → creates → ResultSet     (close in REVERSE)
```
Use **try-with-resources**: all three are `AutoCloseable`, and they're closed in reverse declaration order, even on exceptions. Leaks lead to "Too many connections", exhausted pools, and open transactions.
With a pool, `close()` **returns** the connection to the pool instead of physically closing it.

## ⚠️ Issues in the lecture code (spot-the-bug)
1. `completeCRUD()` doesn't use try-with-resources: `Statement` and `ResultSet` are never closed, and `connection.close()` is skipped on an exception, so connections leak.
2. Every `SQLException` prints **"Database connection failed"**, even a duplicate email or a syntax error. That's misleading.
3. Methods print instead of returning (`getStudent()` builds a list and discards it; `void getStudentById`). They're not reusable and errors are swallowed.
4. Credentials are hardcoded in the repository. Use env vars or config.
5. `getInt("age")` turns SQL `NULL` into `0`.

## 9. Common setup errors
| Error | Fix |
|---|---|
| `Communications link failure` / connection refused | Start MySQL; check host/port 3306/firewall |
| `Access denied for user 'root'@'localhost'` | Wrong password or user grants |
| `Unknown database 'student_db'` | `CREATE DATABASE` first |
| `Public Key Retrieval is not allowed` (DBeaver) | Driver property `allowPublicKeyRetrieval=true` (local dev only) |
| Duplicate entry for key 'email' | UNIQUE constraint working as intended; handle it in the app |

---

## Interview Q&A
**Q1. What is JDBC? JDBC API vs driver?**
JDBC is Java's standard API for relational DBs (`java.sql`). The driver is the vendor implementation that speaks the DB protocol.

**Q2. `Statement` vs `PreparedStatement`?**
`PreparedStatement` uses parameter placeholders: safe from SQL injection, handles types and quoting, and can be pre-compiled and reused. Prefer it for any dynamic value.

**Q3. `executeQuery` vs `executeUpdate` vs `execute`?**
ResultSet for SELECT / affected-row count for DML / boolean telling which kind of result comes first.

**Q4. Where is the ResultSet cursor initially?**
Before the first row. Call `next()` to move to it.

**Q5. Is `Class.forName` needed?**
No, since JDBC 4.0 drivers auto-register via `META-INF/services`.

**Q6. Can PreparedStatement parameterize a column name?**
No, only values. Use allow-list validation for identifiers.

**Q7. Why close JDBC resources? How?**
They hold DB sessions, cursors, and memory. Use try-with-resources, which closes them in reverse order.

**Q8. How do you get an auto-generated key?**
`prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)` then `getGeneratedKeys()`.

**Q9. What are the problems with raw JDBC?**
Boilerplate (connections, statements, closing), checked `SQLException`, manual mapping, no pooling. That leads to Spring JDBC and JPA.

## Quick revision
- App → JDBC API → driver → DB.
- `Connection` (session) → `PreparedStatement` (`?` params, 1-based) → `ResultSet` (cursor before first row).
- `executeQuery` = rows; `executeUpdate` = count.
- Never concatenate user input into SQL.
- try-with-resources always; separate "not found" from "DB error".
