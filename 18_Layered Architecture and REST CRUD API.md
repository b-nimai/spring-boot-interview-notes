# Layered Architecture and REST CRUD API

> **Source:** Lecture 11 (`Notes_Of_Aditya/Lecture 11/code/crudSpringBootDemo`) and Lecture 12 (controller part)
> **Related:** [19_Spring Data JPA](19_Spring%20Data%20JPA%20-%20Entity%2C%20Repository%20and%20Database%20Config.md) · [25_Exception Handling](25_Exception%20Handling%20and%20Global%20Exception%20Handler.md)

## Why this topic matters
CRUD is the skeleton of almost every backend (YouTube videos, Instagram posts, students…). Interviewers expect you to explain **why** code is split into Controller → Service → Repository, and to map CRUD onto HTTP methods correctly.

---

## 1. CRUD ↔ HTTP methods (REST style)
| Operation | HTTP method | REST-style URL | Typical success status |
|---|---|---|---|
| Create | `POST` | `/api/students` | **201 Created** |
| Read all | `GET` | `/api/students` | 200 OK |
| Read one | `GET` | `/api/students/{id}` | 200 OK |
| Update (full) | `PUT` | `/api/students/{id}` | 200 OK |
| Update (partial) | `PATCH` | `/api/students/{id}` | 200 OK |
| Delete | `DELETE` | `/api/students/{id}` | **204 No Content** (or 200 + message) |

> ⚠️ The lecture code uses verb URLs like `/create`, `/get?id=1`, `/update?id=1`, `/delete?id=1`. They work, but **REST convention is nouns in the URL and the verb in the HTTP method**: `POST /api/students`, `GET /api/students/1`. Lecture 16 moves to this style. In an interview, always propose the REST style.

## 2. Why layers?
If one `@RestController` receives the request, validates it, runs the business rules, writes SQL, and builds the response, it becomes unmaintainable very quickly.

```
Postman / React / Mobile app
        ↓  HTTP request
   Controller   → receive request, read input, call service, return response
        ↓
    Service     → business rules (exists? soft or hard delete? duplicate email?)
        ↓
   Repository   → talk to the database (save, findById, delete…)
        ↓
    Database (MySQL)
```
The response travels back up the same path.
**Entity / Model** isn't a layer. It's the object that **travels between** the layers (`Student`).

| Layer | Annotation | Responsibility |
|---|---|---|
| Controller | `@RestController` | HTTP in/out only |
| Service | `@Service` | Business logic |
| Repository | `@Repository` / `JpaRepository` | DB operations |
| Entity | `@Entity` | Java ↔ table mapping |

### Package structure
```
in.strikes.crudSpringBootDemo
├── CrudSpringBootDemoApplication.java   ← root package (component scan starts here)
├── controller/StudentController.java
├── service/StudentService.java
├── repository/StudentRepository.java
└── entity/Student.java
```

## 3. Lecture 11: tracing the flow (fake repository)
```java
@RestController
@RequestMapping("/api/students")
public class StudentController {
    private StudentService studentService;
    public StudentController(StudentService studentService) { this.studentService = studentService; }

    @PostMapping("/create")
    public ResponseEntity<Student> createStudent(@RequestBody Student student) {
        System.out.println("Inside Student Controller");
        Student createdStudent = studentService.createStudent(student);
        System.out.println("Exiting Student Controller");
        return ResponseEntity.status(HttpStatus.CREATED).body(createdStudent);
    }
}

@Service
public class StudentService {
    public Student createStudent(Student req) {
        System.out.println("Inside Student Service");
        Student resp = studentRepository.saveStudent(req);
        System.out.println("Exiting Student Service");
        return resp;
    }
}

@Repository
public class StudentRepository {
    public Student saveStudent(Student req) {
        System.out.println("Inside Student Repository");
        System.out.println("Exiting Student Repository");
        Student s1 = new Student(); s1.setName("Aditya"); /* ...hardcoded... */
        return s1;
    }
}
```
Console for `POST /api/students/create`:
```
Inside Student Controller
Inside Student Service
Inside Student Repository
Exiting Student Repository
Exiting Student Service
Exiting Student Controller
```
This is the same **constructor DI** from Lectures 4–5, now inside a real web app.

## 4. Dependencies (Spring Initializr)
| Dependency | Why |
|---|---|
| Spring Web (`spring-boot-starter-webmvc` in Boot 4, `-starter-web` in Boot 3) | Embedded Tomcat, DispatcherServlet, `@RestController`, Jackson JSON |
| Spring Data JPA | Hibernate, `JpaRepository`, no hand-written SQL for basic CRUD |
| MySQL Driver (`mysql-connector-j`) | JDBC driver so Java can talk to MySQL |

### ⚠️ Classic startup failure
Adding JPA + MySQL driver **without** `spring.datasource.*` properties makes the app crash:
```
Failed to configure a DataSource: 'url' attribute is not specified
Reason: Failed to determine a suitable driver class
```
Why: auto-configuration sees JPA and the driver on the classpath, so it tries to build a `DataSource` → `EntityManagerFactory`. With no URL, it fails.
Fixes: add DB properties, or temporarily `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)` (Boot 4 package: `org.springframework.boot.jdbc.autoconfigure`).

## 5. Lecture 12 controller: full CRUD with `ResponseEntity`
```java
@GetMapping("/get")
public ResponseEntity<Student> getStudent(@RequestParam Long id) {        // /get?id=1
    Student s = studentService.getStudent(id);
    if (s == null) return ResponseEntity.notFound().build();              // 404, empty body
    return ResponseEntity.ok(s);                                          // 200 + JSON
}

@PutMapping("/update")
public ResponseEntity<Student> updateStudent(@RequestParam Long id, @RequestBody Student req) { ... }

@DeleteMapping("/delete")
public ResponseEntity<String> deleteStudent(@RequestParam Long id) {
    if (!studentService.deleteStudent(id)) return ResponseEntity.notFound().build();
    return ResponseEntity.ok("Record deleted");
}
```

### Request-data annotations
| Annotation | Reads from | Example |
|---|---|---|
| `@RequestBody` | JSON body → Java object (Jackson) | `POST` body `{ "name": "Rahul" }` |
| `@RequestParam` | Query string | `/get?id=1` |
| `@PathVariable` | URL path segment | `/students/{id}` → `/students/1` |
| `@RequestHeader` | HTTP header | `@RequestHeader("token") String t` |

### `ResponseEntity` = status + headers + body
```java
ResponseEntity.ok(body);                                     // 200
ResponseEntity.status(HttpStatus.CREATED).body(saved);       // 201
ResponseEntity.created(URI.create("/api/students/" + id)).body(saved); // 201 + Location header
ResponseEntity.noContent().build();                          // 204
ResponseEntity.notFound().build();                           // 404, no body
```
Without `ResponseEntity`, Spring always sends 200, and returning `null` produces an empty 200, which is misleading.

---

## Common pitfalls
- **Main class not in the root package** means controllers aren't scanned, so you get 404.
- Returning **404 for an empty list** (`getAll` in the lecture) is debatable. An empty collection is usually `200 []`, because the *collection* exists.
- Returning `null` from the service and null-checking in every controller leads to duplicated code. Lecture 17 replaces this with custom exceptions + `@RestControllerAdvice`.
- Returning the **entity directly** exposes internal fields. Lecture 16 introduces DTOs.

## Interview Q&A
**Q1. Why do we use a layered architecture?**
Separation of concerns: the controller handles HTTP, the service handles business rules, and the repository handles persistence. Each layer is testable in isolation and easier to change (for example, swapping the DB doesn't touch the controller).

**Q2. Can the controller call the repository directly?**
Technically yes, but then business logic ends up in the controller and can't be reused. Keep the controller thin.

**Q3. `@RequestParam` vs `@PathVariable`?**
`@PathVariable` identifies a resource (`/students/5`). `@RequestParam` filters or modifies a query (`/students?city=Delhi&page=2`).

**Q4. Why return `ResponseEntity`?**
To control the status code, headers (e.g. `Location`), and body explicitly, e.g. 201 on create and 404 when not found.

**Q5. Which status code for create and for delete?**
201 Created (ideally with a `Location` header) and 204 No Content.

**Q6. Who converts JSON to `Student`?**
Jackson, through Spring MVC's `HttpMessageConverter`, triggered by `@RequestBody` (and `@RestController` for responses).

**Q7. Why did the app fail on startup after adding JPA + MySQL?**
DataSource auto-configuration kicked in because of classpath detection, but no `spring.datasource.url` was set.

## Quick revision
- Controller → Service → Repository → DB; the entity travels between layers.
- REST: nouns in URLs, verbs as HTTP methods.
- `@RequestBody` = JSON body, `@RequestParam` = query, `@PathVariable` = path.
- `ResponseEntity` controls status (201 create, 204 delete, 404 missing).
