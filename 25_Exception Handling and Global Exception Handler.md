# Exception Handling and the Global Exception Handler

> **Source:** Lecture 17 (`Notes_Of_Aditya/Lecture 17/code/exceptionDemo`; same code as Lecture 16's `dtoDemo`)
> **Related:** [24_DTOs and Bean Validation](24_DTOs%20and%20Bean%20Validation.md) · [18_Layered Architecture](18_Layered%20Architecture%20and%20REST%20CRUD%20API.md)

## Why this topic matters
A professional API tells the client **what happened** with the right status code and a consistent error body. `@RestControllerAdvice` + `@ExceptionHandler` + custom exceptions is asked about in nearly every Spring Boot interview.

---

## 1. Status codes for CRUD APIs
| Situation | Code |
|---|---|
| Fetched / updated OK | **200 OK** |
| Created | **201 Created** (+ `Location` header) |
| Deleted, no body | **204 No Content** |
| Validation failed, malformed JSON, wrong type | **400 Bad Request** |
| Not authenticated | 401 Unauthorized |
| Authenticated but not allowed | 403 Forbidden |
| Resource doesn't exist | **404 Not Found** |
| Duplicate email/username, state conflict | **409 Conflict** |
| Unexpected bug or DB down | **500 Internal Server Error** |

> **400 vs 409:** 400 = the request itself is invalid (bad email format, `"age": "twenty"`). 409 = the request is valid but conflicts with existing data (email already exists).
> **404 vs 500:** "student not found" is a normal business case, so it's **404**, not 500.

## 2. Beginner problems this solves
1. Everything returns 200 (even "not found" → 200 with `null`).
2. `if (x == null) return 404` repeated in every controller method.
3. Inconsistent error formats across endpoints.
4. Unhandled exceptions end up in Spring Boot's default `/error` → **Whitelabel Error Page** (browser) or:
```json
{ "timestamp": "...", "status": 500, "error": "Internal Server Error", "path": "/test/500" }
```
The exception message is hidden by default. `spring.web.error.include-message=always` shows it (for local debugging only; it's used in the lecture's `application.properties`).

## 3. Clean responsibility split
| Layer | Does |
|---|---|
| Controller | Receives request, calls service, returns the **success** response |
| Service | Business checks; **throws meaningful exceptions** |
| `@RestControllerAdvice` | Converts exceptions into HTTP error responses, in **one place** |

```
Controller → Service ──throws ResourceNotFoundException──►
   DispatcherServlet → ExceptionHandlerExceptionResolver → GlobalExceptionHandler → 404 JSON
```

## 4. Custom exceptions
```java
public class ResourceNotFoundException extends RuntimeException {      // unchecked: no 'throws' everywhere
    public ResourceNotFoundException(String message) { super(message); }
}
public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String message) { super(message); }
}
```
Service: **throw instead of returning null**:
```java
public CreateStudentResponseDto getStudent(Long id) {
    Student s = studentRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Student with id " + id + " not found"));
    return mapToDto(s);
}

public void deleteStudent(Long id) {
    Student s = studentRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Student with id " + id + " not found"));
    studentRepository.delete(s);
}
```
The controller becomes clean, with no `if`s:
```java
@GetMapping("/{id}")
public ResponseEntity<CreateStudentResponseDto> getStudent(@PathVariable Long id) {
    return ResponseEntity.ok(studentService.getStudent(id));
}

@DeleteMapping
public ResponseEntity<String> deleteStudent(@RequestParam Long id) {
    studentService.deleteStudent(id);
    return ResponseEntity.noContent().build();          // 204
}
```

## 5. Standard error DTO
```java
public class ExceptionResponseDto {
    private LocalDateTime timestamp;   // match with server logs
    private int statusCode;            // frontend can branch on it
    private String error;              // "Not Found"
    private String message;            // "Student with id 100 not found"
    private String path;               // which endpoint failed
    // constructor + getters
}
```
```json
{
  "timestamp": "2026-07-07T10:45:30",
  "statusCode": 404,
  "error": "Not Found",
  "message": "Student with id 100 not found",
  "path": "/api/students/100"
}
```

## 6. Global handler (lecture code)
```java
@RestControllerAdvice                            // = @ControllerAdvice + @ResponseBody
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ExceptionResponseDto> handleResourceNotFoundException(
            ResourceNotFoundException ex, HttpServletRequest request) {
        ExceptionResponseDto body = new ExceptionResponseDto(
                LocalDateTime.now(),
                HttpStatus.NOT_FOUND.value(),
                HttpStatus.NOT_FOUND.getReasonPhrase(),   // "Not Found"
                ex.getMessage(),
                request.getRequestURI());                 // path
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(body);
    }

    @ExceptionHandler(DuplicateResourceException.class)  // → 409 CONFLICT (same shape)
    ...

    @ExceptionHandler(MethodArgumentNotValidException.class)   // @Valid failed → 400 + fieldErrors
    public ResponseEntity<ValidationExceptionResponseDto> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest request) {
        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(err -> fieldErrors.put(err.getField(), err.getDefaultMessage()));
        return ResponseEntity.badRequest().body(new ValidationExceptionResponseDto(
                LocalDateTime.now(), 400, "Bad Request", "Validation Failed",
                request.getRequestURI(), fieldErrors));
    }

    @ExceptionHandler(RuntimeException.class)   // → 500
    ...
    @ExceptionHandler(Exception.class)          // safety net → 500
    ...
}
```

### Which handler wins?
Spring picks the **most specific** matching exception type (closest in the class hierarchy).
`ResourceNotFoundException` goes to its own handler, **not** `RuntimeException` or `Exception`, even though those also match.

### More 400 handlers worth adding (from the notes)
| Exception | When | Status |
|---|---|---|
| `HttpMessageNotReadableException` | Malformed JSON, `"age": "twenty"` | 400 |
| `MethodArgumentTypeMismatchException` | `GET /api/students/abc` (Long expected) | 400 |
| `MissingServletRequestParameterException` | Required `@RequestParam` missing | 400 |
| `HttpRequestMethodNotSupportedException` | POST to a GET-only URL | 405 |
| `NoResourceFoundException` (Boot 3.2+) | Unknown URL | 404 |

```java
@ExceptionHandler(MethodArgumentTypeMismatchException.class)
public ResponseEntity<ExceptionResponseDto> handleTypeMismatch(MethodArgumentTypeMismatchException ex, HttpServletRequest req) {
    return ResponseEntity.badRequest().body(new ExceptionResponseDto(LocalDateTime.now(), 400, "Bad Request",
            "Invalid value for parameter '" + ex.getName() + "'. Please check the required data type.",
            req.getRequestURI()));
}
```

## 7. ResponseEntity recap
```java
ResponseEntity.ok(dto);                                    // 200
ResponseEntity.status(HttpStatus.CREATED).body(dto);       // 201
ResponseEntity.created(URI.create("/api/students/" + id)).body(dto);  // 201 + Location header
ResponseEntity.noContent().build();                        // 204
ResponseEntity.badRequest().body(err);                     // 400
```

## 8. Best practices
- ❌ `findById(id).orElse(null)` → ✅ `orElseThrow(() -> new ResourceNotFoundException(...))`
- Success responses go in the controller, business checks in the service, error responses in the advice.
- Specific handlers first; the `Exception` handler is only a safety net.
- **Never return raw `ex.getMessage()` for unexpected errors** (it can leak SQL, class names, stack details). Log the exception and return `"Something went wrong. Please try again later."`
- **Log** unexpected exceptions (`log.error("...", ex)`). Otherwise the safety net silently swallows bugs.
- Modern alternative: Spring 6+ supports **RFC 7807 `ProblemDetail`** (`application/problem+json`). Return `ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, msg)`, or extend `ResponseEntityExceptionHandler`.

> ⚠️ **In the lecture code**, the `RuntimeException` and `Exception` handlers return `ex.getMessage()` directly. That's fine for learning, but a production leak risk. Also note that `@ResponseStatus(HttpStatus.NOT_FOUND)` on the exception class is a lighter alternative when you don't need a custom body.

---

## Interview Q&A
**Q1. How do you handle exceptions globally in Spring Boot?**
Create a `@RestControllerAdvice` class with `@ExceptionHandler` methods that map exception types to `ResponseEntity` error bodies.

**Q2. `@ControllerAdvice` vs `@RestControllerAdvice`?**
`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`, so handler return values are written as JSON instead of being resolved as views.

**Q3. If handlers exist for `Exception` and `ResourceNotFoundException`, which runs?**
The most specific one: `ResourceNotFoundException`.

**Q4. Local `@ExceptionHandler` in a controller vs global advice: which wins?**
The controller-local handler takes precedence for exceptions thrown by that controller.

**Q5. Why should custom exceptions extend `RuntimeException`?**
They're unchecked, so there are no `throws` clauses everywhere. They also trigger transaction rollback by default with `@Transactional`.

**Q6. Which exception does `@Valid` raise? How do you return field errors?**
`MethodArgumentNotValidException`. Iterate `ex.getBindingResult().getFieldErrors()` into a `Map<field, message>`.

**Q7. When 400 vs 409 vs 404?**
400: malformed or invalid input. 409: valid input that conflicts with current state (duplicate). 404: resource doesn't exist.

**Q8. What is the Whitelabel Error Page?**
Spring Boot's default HTML error view from `BasicErrorController` at `/error`, shown when nothing else handles the error.

**Q9. Can `@ControllerAdvice` handle exceptions thrown in a Filter?**
No. Filters run before DispatcherServlet. Handle errors inside the filter, or delegate to a `HandlerExceptionResolver`.

## Quick revision
- Service throws custom exceptions; the advice maps them to status + standard body.
- Most specific `@ExceptionHandler` wins; `Exception` is the fallback.
- 201 create, 204 delete, 400 bad input, 404 missing, 409 duplicate, 500 bug.
- Don't leak `ex.getMessage()` for unexpected errors; log them instead.
