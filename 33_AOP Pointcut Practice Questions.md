# AOP Pointcut Practice Questions

> **Source:** `Notes_Of_Aditya/Lecture 24/PracticeQuestions.pdf` (questions + answer key), rewritten with explanations.
> **Theory:** [32_AOP Pointcut Expressions](32_AOP%20Pointcut%20Expressions.md)
> Try each one first, then expand **Answer**.

## Setup
```
com.coderarmy.student
├── controller   StudentController, AdminController
├── service      StudentService, EmailService
│   └── impl     PaymentServiceImpl
├── repository   StudentRepository
└── model        Student, PaymentRequest
```
```java
package com.coderarmy.student.service;
public class StudentService {
    public Student getStudent(Long id)
    public List<Student> getAllStudents()
    public Student createStudent(Student student)
    public Student updateStudent(Long id, Student student)
    public void deleteStudent(Long id)
    public boolean studentExists(String email)
    public void clearCache()
}
```

---

## Section 1: basic `execution()`

**Q1.** Match every interceptable method.
<details><summary>Answer</summary>

`execution(* *(..))`: any return type, any name, any params.
</details>

**Q2.** Every method named `getStudent` (any class, return type, params).
<details><summary>Answer</summary>

`execution(* getStudent(..))`. ⚠️ This also matches `StudentController.getStudent`. Add a class or package to narrow it.
</details>

**Q3.** Names beginning with `get`.
<details><summary>Answer</summary>

`execution(* get*(..))`
</details>

**Q4.** Names ending with `Student` (`getStudent`, `createStudent`… but not `getStudentById`).
<details><summary>Answer</summary>

`execution(* *Student(..))`
</details>

**Q5.** Names containing `Student`.
<details><summary>Answer</summary>

`execution(* *Student*(..))`
⚠️ The PDF says it should also match `studentExists()`. It **won't**: matching is **case-sensitive**, and `studentExists` has a lowercase `s`. To cover both you'd need `execution(* *Student*(..)) || execution(* student*(..))`.
</details>

**Q6.** Exactly zero parameters.
<details><summary>Answer</summary>

`execution(* *())`
</details>

**Q7.** Exactly one parameter of any type.
<details><summary>Answer</summary>

`execution(* *(*))`
</details>

**Q8.** Exactly one `Long` parameter.
<details><summary>Answer</summary>

`execution(* *(java.lang.Long))`. `updateStudent(Long, Student)` doesn't match (two params).
</details>

## Section 2: intermediate `execution()`

**Q9.** Every method in `StudentService`.
<details><summary>Answer</summary>

`execution(* com.coderarmy.student.service.StudentService.*(..))`
</details>

**Q10.** Only `getStudent(Long)` in `StudentService`.
<details><summary>Answer</summary>

`execution(* com.coderarmy.student.service.StudentService.getStudent(java.lang.Long))`
</details>

**Q11.** All public methods in `StudentService`.
<details><summary>Answer</summary>

`execution(public * com.coderarmy.student.service.StudentService.*(..))`
</details>

**Q12.** Classes **directly** in `service` (not `service.impl`).
<details><summary>Answer</summary>

`execution(* com.coderarmy.student.service.*.*(..))`. A single `.` followed by `*` means one level only.
</details>

**Q13.** `service` **and all sub-packages**.
<details><summary>Answer</summary>

`execution(* com.coderarmy.student.service..*.*(..))`
</details>

**Q14.** Public methods starting with `get`, in `service` or its sub-packages.
<details><summary>Answer</summary>

`execution(public * com.coderarmy.student.service..*.get*(..))`. This matches `PaymentServiceImpl.getPaymentStatus` but not `StudentController.getStudent`.
</details>

**Q15.** First parameter is `Long`.
<details><summary>Answer</summary>

`execution(* *(java.lang.Long, ..))`
</details>

**Q16.** Methods returning `void`.
<details><summary>Answer</summary>

`execution(void *(..))`
</details>

## Section 3: `within()` and annotations

**Q17.** Service layer using only `within()`.
<details><summary>Answer</summary>

`within(com.coderarmy.student.service..*)`
</details>

**Q18.** Public `get*` methods, service layer only, combining `within` + `execution`.
<details><summary>Answer</summary>

`within(com.coderarmy.student.service..*) && execution(public * get*(..))`
</details>

**Q19.** Methods in classes annotated `@Audited` (`com.coderarmy.student.annotation.Audited`).
<details><summary>Answer</summary>

`@within(com.coderarmy.student.annotation.Audited)`
</details>

**Q20.** Methods annotated `@TrackExecutionTime`.
<details><summary>Answer</summary>

`@annotation(com.coderarmy.student.annotation.TrackExecutionTime)`
</details>

## Tricky questions

**A.** Service layer except `clearCache()`.
<details><summary>Answer</summary>

`within(com.coderarmy.student.service..*) && !execution(* clearCache())`. Use `clearCache(..)` to exclude every overload.
</details>

**B.** Controllers **or** services (incl. sub-packages).
<details><summary>Answer</summary>

`within(com.coderarmy.student.controller..*) || within(com.coderarmy.student.service..*)`
</details>

**C.** Public methods in the whole app except repositories.
<details><summary>Answer</summary>

`within(com.coderarmy.student..*) && execution(public * *(..)) && !within(com.coderarmy.student.repository..*)`
</details>

**D.** The bean named `studentService`.
<details><summary>Answer</summary>

`bean(studentService)`
</details>

**E.** Beans whose names end with `Service`.
<details><summary>Answer</summary>

`bean(*Service)`. Bean names are case-sensitive.
</details>

**F.** Methods receiving a `Student` at runtime.
<details><summary>Answer</summary>

`args(com.coderarmy.student.model.Student)` matches exactly one argument. For "first argument is Student, maybe more": `args(com.coderarmy.student.model.Student, ..)`.
</details>

**G.** Runtime argument class annotated `@SensitiveData`.
<details><summary>Answer</summary>

`@args(com.coderarmy.student.annotation.SensitiveData)`. It checks the **argument's class**, not a parameter annotation.
</details>

**H.** The actual bean behind the proxy is `StudentService`.
<details><summary>Answer</summary>

`target(com.coderarmy.student.service.StudentService)`
</details>

**I.** The proxy implements `StudentOperations`.
<details><summary>Answer</summary>

`this(com.coderarmy.student.service.StudentOperations)`
</details>

**J. Parameter drills**
<details><summary>Answers</summary>

| Requirement | Expression |
|---|---|
| J1 zero params | `execution(* *())` |
| J2 exactly one | `execution(* *(*))` |
| J3 zero or more | `execution(* *(..))` |
| J4 exactly two | `execution(* *(*,*))` |
| J5 first is Long | `execution(* *(java.lang.Long, ..))` |
| J6 last is Student | `execution(* *(.., com.coderarmy.student.model.Student))` |
</details>

## Challenge questions

**C1.** Public service methods except those starting with `delete`.
<details><summary>Answer</summary>

`within(com.coderarmy.student.service..*) && execution(public * *(..)) && !execution(* delete*(..))`
</details>

**C2.** `get*` methods with no parameters.
<details><summary>Answer</summary>

`execution(* get*())`
</details>

**C3.** Methods returning exactly `Student`.
<details><summary>Answer</summary>

`execution(com.coderarmy.student.model.Student *(..))`. It doesn't match `List<Student>`, `Optional<Student>`, `void`, or `boolean`.
</details>

**C4.** Exactly `(Long, Student)` in that order.
<details><summary>Answer</summary>

`execution(* *(java.lang.Long, com.coderarmy.student.model.Student))`. `someMethod(Student, Long)` doesn't match (order matters).
</details>

**C5.** Public service-layer methods in classes annotated `@Audited`.
<details><summary>Answer</summary>

`within(com.coderarmy.student.service..*) && execution(public * *(..)) && @within(com.coderarmy.student.annotation.Audited)`
</details>

---

## Wildcard cheat sheet
| Pattern | Meaning |
|---|---|
| `*` return type / name | any |
| `get*` · `*Student` · `*Student*` | starts / ends / contains (case-sensitive) |
| `()` · `(*)` · `(*,*)` · `(..)` | 0 · 1 · 2 · any params |
| `(Long, ..)` · `(.., Student)` | first is Long · last is Student |
| `service.*` · `service..*` | direct package · package + sub-packages |
