# DTOs and Bean Validation

> **Source:** Lecture 16 (`Notes_Of_Aditya/Lecture 16/code/dtoDemo`)
> **Related:** [18_Layered Architecture](18_Layered%20Architecture%20and%20REST%20CRUD%20API.md) · [25_Exception Handling](25_Exception%20Handling%20and%20Global%20Exception%20Handler.md)

## Why this topic matters
"Why not return the entity directly?" and "How do you validate request input?" are standard interview questions. DTOs + validation are what separate a demo API from a production API.

---

## Part A: DTOs

### 1. The problem: one entity used everywhere
Using `Student` (the `@Entity`) as the request, the response, **and** the DB model mixes two worlds:
```
Outside world (API contract)   ←→   Inside world (DB tables, business rules)
```
| Problem | Example |
|---|---|
| **Client sets internal fields** | Sends `"id": 100`, `"deleted": true`, `"createdAt": ...` |
| **Mass assignment** (security) | Sends `"role": "ADMIN"`, `"isVerified": true` on update |
| **Data leakage** | Response exposes `password`, `deleted`, `internalRemark` |
| **Request ≠ response shape** | Request needs no `id`; response needs `id` + `message` |
| **DB change breaks API** | Split `name` → `firstName`/`lastName` and the frontend breaks |
| (JPA-specific) | Lazy relations → `LazyInitializationException`; bidirectional relations → infinite JSON recursion |

### 2. DTO = Data Transfer Object
A plain class that carries **only the data needed for one request or response**. It has no JPA annotations.

| Entity | DTO |
|---|---|
| DB mapping (inside) | API contract (outside) |
| Has internal fields (`deleted`, `createdAt`…) | Only allowed or needed fields |
| Client never controls it | Client controls it in a limited way |

Lecture DTOs (one per use case):
```
dto/
├── CreateStudentRequestDto     ← name, age, email, rollNo, subject (+ validation)
├── CreateStudentResponseDto    ← id, …, message, createdAt, updatedAt
├── UpdateStudentRequestDto     ← name, age, rollNo, subject  (NO email: can't change it)
├── UpdateStudentResponseDto    ← id, …, message, updatedAt
├── ExceptionResponseDto        ← timestamp, statusCode, error, message, path
└── ValidationExceptionResponseDto ← … + Map<String,String> fieldErrors
```
> **Use-case-specific DTOs are the strength here:** the update DTO simply doesn't have `email`, so the client *cannot* change it.

### 3. Mapping (in the Service layer)
```
Controller ──DTO──► Service ──Entity──► Repository
Controller ◄─DTO─── Service ◄─Entity─── Repository
```
The controller never touches the entity, and the repository never touches DTOs.
```java
private Student mapToEntity(CreateStudentRequestDto dto) {
    Student s = new Student();
    s.setName(dto.getName());
    s.setAge(dto.getAge());
    s.setEmail(dto.getEmail());
    s.setRollNo(dto.getRollNo());
    s.setSubject(dto.getSubject());
    s.setCreatedAt(LocalDateTime.now());   // backend-controlled fields
    s.setUpdatedAt(LocalDateTime.now());
    s.setDeleted(false);
    return s;
}

private CreateStudentResponseDto mapToDto(Student s) {
    CreateStudentResponseDto r = new CreateStudentResponseDto();
    r.setId(s.getId());
    r.setName(s.getName()); /* … */
    r.setMessage("Student saved successfully");
    r.setCreatedAt(s.getCreatedAt());
    r.setUpdatedAt(s.getUpdatedAt());
    return r;
}

public CreateStudentResponseDto createStudent(CreateStudentRequestDto dto) {
    Student student = mapToEntity(dto);
    if (studentRepository.existsByEmail(student.getEmail()))
        throw new DuplicateResourceException("Student with email " + student.getEmail() + " already exists");
    return mapToDto(studentRepository.save(student));
}

public List<CreateStudentResponseDto> getAllStudent() {
    return studentRepository.findByDeletedIsFalse().stream().map(this::mapToDto).toList();
}
```
Larger projects move this to a `StudentMapper` class or use **MapStruct** (compile-time generated mappers). Java `record`s make concise immutable DTOs:
```java
public record StudentResponse(Long id, String name, String email) {}
```

## Part B: Bean Validation

### 4. Setup
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>   <!-- Hibernate Validator (Jakarta Bean Validation) -->
</dependency>
```

### 5. Rules on the request DTO
```java
public class CreateStudentRequestDto {
    @NotBlank(message = "Name cannot be null/Empty or blank")
    @Size(min = 2, max = 50, message = "Student name must be within 2 to 50 character long")
    private String name;

    @NotNull(message = "Age is required")
    @Min(value = 18, message = "Student must be atleast 18 years old")
    private Integer age;                       // wrapper type so "missing" = null, not 0

    @NotBlank(message = "Student email cannot be blank")
    @Email(message = "Student email must be valid")
    private String email;

    @NotNull(message = "RollNo is required")
    private Integer rollNo;

    @NotBlank(message = "Subject is required")
    private String subject;
}
```

### 6. Trigger it with `@Valid`
```java
@PostMapping
public ResponseEntity<CreateStudentResponseDto> createStudent(
        @Valid @RequestBody CreateStudentRequestDto dto) {        // annotations = rules, @Valid = run them
    return ResponseEntity.status(HttpStatus.CREATED).body(studentService.createStudent(dto));
}
```
Flow:
```
JSON → (Jackson) DTO → @Valid checks → ❌ invalid: MethodArgumentNotValidException → 400 (service never runs)
                                      → ✅ valid: service → DB
```

### 7. Annotation cheat sheet
| Annotation | For | Rejects |
|---|---|---|
| `@NotNull` | any type | `null` |
| `@NotEmpty` | String/Collection/Array | `null`, `""`, `[]` |
| `@NotBlank` | String | `null`, `""`, `"   "` |
| `@Size(min,max)` | String/Collection | length or size out of range |
| `@Min` / `@Max` | numbers | below/above |
| `@Positive` / `@PositiveOrZero` | numbers | ≤ 0 / < 0 |
| `@Email` | String | bad format (note: `null` passes, so combine with `@NotBlank`) |
| `@Pattern(regexp)` | String | non-matching (e.g. 10-digit mobile `^[6-9]\\d{9}$`) |
| `@Past` / `@Future` | dates | wrong side of now |
| `@Valid` on a field | nested object / list | cascades validation into it |

> **@NotNull vs @NotEmpty vs @NotBlank:** for `" "`, `@NotNull` ✅ passes, `@NotEmpty` ✅ passes, `@NotBlank` ❌ fails. Use `@NotBlank` for text and `@NotNull` for numbers and objects.

### 8. Validation response (with the handler from Lecture 17)
```json
{
  "timestamp": "2026-07-07T11:20:30",
  "statusCode": 400,
  "error": "Bad Request",
  "message": "Validation Failed",
  "path": "/api/students",
  "fieldErrors": {
    "name": "Name cannot be null/Empty or blank",
    "email": "Student email must be valid",
    "age": "Student must be atleast 18 years old"
  }
}
```

---

## ⚠️ Issues in the lecture code (spot-the-bug practice)
1. **`UpdateStudentRequestDto` has no validation annotations, and the PUT endpoint has no `@Valid`**, so blank names can be saved through update.
2. The update DTO uses primitive `int age` / `int rollNo`: a missing field silently becomes `0`. Prefer `Integer` + `@NotNull`.
3. `CreateStudentRequestDto.getAge()` returns `int` from an `Integer` field. Auto-unboxing a `null` gives a `NullPointerException`. It's safe here only because `@Valid` rejects `null` first.
4. `getStudent()` and `deleteStudent()` use `findById`, so soft-deleted students are still visible or deletable. Use `findByIdAndDeletedIsFalse`.
5. `existsByEmail` then `save` is a check-then-act race. Two concurrent requests can both pass. Add a **DB unique constraint** (`@Column(unique = true)`) as the real guard.

## Interview Q&A
**Q1. Why use DTOs instead of exposing entities?**
To decouple the API contract from the DB schema, prevent mass assignment, avoid leaking internal or sensitive fields, shape request and response separately, and avoid JPA serialization issues (lazy loading, recursion).

**Q2. What is mass assignment?**
When the client sends extra fields (e.g. `role=ADMIN`) and the server binds them blindly onto the domain object. DTOs with only allowed fields prevent it.

**Q3. Where should mapping happen?**
In the service layer (or a dedicated mapper/MapStruct). The controller works with DTOs and the repository with entities.

**Q4. `@Valid` vs `@Validated`?**
`@Valid` (Jakarta) triggers validation and cascades into nested objects. `@Validated` (Spring) also supports **validation groups** and enables method-level validation on beans (e.g. `@Min` on `@RequestParam` in a `@Validated` controller).

**Q5. Which exception is thrown when `@Valid @RequestBody` fails? Status?**
`MethodArgumentNotValidException`, returned as 400 Bad Request.

**Q6. Should validation go on entities or DTOs?**
Request-input validation belongs on the request DTO. DB constraints (`nullable`, `unique`) on the entity and schema are the last line of defense.

**Q7. Why use `Integer` instead of `int` in request DTOs?**
To tell "not sent" (`null`) apart from `0`, so `@NotNull` can work.

## Quick revision
- Entity = inside (DB), DTO = outside (API). Never expose entities.
- Use one DTO per use case (create vs update).
- Map in the service: DTO → Entity → save → Entity → DTO.
- Rules on the DTO + `@Valid` in the controller → 400 with field errors.
- `@NotBlank` for strings, `@NotNull` for numbers; use wrapper types.
