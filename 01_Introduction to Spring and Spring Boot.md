# Introduction to Spring and Spring Boot

> **Source:** Lecture 01 (`Notes_Of_Aditya/Lecture 01/notes/Notes.pdf`)

## Why this topic matters
If you start directly with `@RestController`, Spring Boot feels like magic. Knowing **which problem each layer solves** (HTTP → Servlets → Spring → Spring Boot) makes debugging and interviews much easier.

---

## 1. Client–Server Architecture
```
Client (Browser / Postman / React / Mobile app)  ---- request ---->  Server (Spring Boot app)
                                                  <--- response ----
```
- **Client** asks for something.
- **Server** receives the request, processes it (auth, DB, business rules), and sends a response.

## 2. HTTP – the language between client and server
HTTP (HyperText Transfer Protocol) is an **application-layer protocol on top of TCP/IP**. It defines what a request and a response look like.

### HTTP Request = Method + Path + Headers + Body
```http
POST /login
Content-Type: application/json

{ "email": "abc@gmail.com", "password": "12345" }
```

| Method | Meaning | Example |
|---|---|---|
| `GET` | Read data | Fetch list of orders |
| `POST` | Create data | Place a new order |
| `PUT` | Replace completely | Update entire profile |
| `PATCH` | Update partially | Change only phone number |
| `DELETE` | Remove data | Cancel an order |

**Headers** = key-value metadata (`Accept`, `Content-Type`, `Authorization: Bearer <token>`, `Host`).
**Body** = actual data (usually JSON) – used with POST/PUT/PATCH. GET usually has no body.

### HTTP Response = Status code + Headers + Body
```http
HTTP/1.1 200 OK
Content-Type: application/json

{ "message": "Login successful" }
```
| Code | Meaning |
|---|---|
| 200 | Success |
| 404 | Resource not found |
| 500 | Internal server error |

## 3. Normal Java program vs Web application
```
Normal program:  Start → Run instructions → Finish → Exit
Web server:      Start → Keep running → Wait for requests → Process → Respond → Keep running
```

## 4. Why Core Java alone is not enough
Java can open a port:
```java
ServerSocket server = new ServerSocket(8080);   // java.net, since Java 1.0
```
But the JVM only gets a **raw stream of bytes**. It does not know that `GET` means fetch or that `/users` is an endpoint. You would have to do all of this yourself:
1. Open the port and read raw input streams
2. Parse HTTP manually (method, URL, headers, body)
3. Route the request yourself: `if (url.equals("/users")) {...} else if (...)`
4. Build the HTTP response format
5. Manage threads for many users, and handle errors

None of this is business logic. It's repeated plumbing work, which is why a standard solution was needed.

## 5. Servlets and Servlet Containers
- **Servlet** = a Java class designed to handle HTTP requests (`doGet()`, `doPost()`).
- **Servlet Container** (Tomcat, Jetty, Undertow) does the low-level work: opens the port, parses HTTP, creates request/response objects, manages threads, calls servlet methods, and sends the response.

```
HTTP Request → Servlet Container (Tomcat) → Servlet → Java code
```

## 6. Why Spring?
Large apps written directly with Servlets had too much boilerplate and configuration, tight coupling, and were hard to test and maintain.
**Spring Framework** introduced **IoC, Dependency Injection, Bean management, and loose coupling**.

### Spring ecosystem
| Module | Purpose |
|---|---|
| **Spring Core** | Foundation: IoC, DI, beans, ApplicationContext |
| **Spring MVC** | Web apps / REST APIs, built on **Servlets + Spring Core** |
| **Spring Data** | DB access. Flow: `Spring Data JPA → Hibernate → JDBC → DB` |
| **Spring Security** | Authentication and authorization: login, JWT, OAuth, roles, CSRF |
| **Spring AOP** | Cross-cutting concerns: logging, transactions, security checks |
| **Spring Boot** | Auto-configuration and quick setup |
| **Spring AI** | Integrating LLMs, embeddings, RAG, vector DBs |

**JPA vs Hibernate:** JPA is a *specification* (it says what should be done). Hibernate is an *implementation* (it does the work).

## 7. Spring Boot
Spring Boot is **not a replacement** for Spring. It is an **automation layer on top of Spring** that provides:
- Auto-configuration
- Starter dependencies
- Embedded server (Tomcat)
- Sensible defaults ("opinionated")
- Production-ready features

**Opinionated** means it assumes sensible defaults. For example, if you add the web dependency, Boot assumes you want a web app and configures embedded Tomcat, Spring MVC, JSON support, and error handling.

| Spring Framework | Spring Boot |
|---|---|
| Core features and modules | Auto-configuration and quick setup |
| More manual configuration | Less manual configuration |
| Flexibility | Sensible defaults |
| Foundation | Built on top of Spring |

**Microservices** = an *architecture style* (independent User/Order/Payment services). Spring Boot is a *tool* commonly used to build them.

## 8. Complete flow
```
Browser sends HTTP request
   ↓
Servlet Container (embedded Tomcat) receives it
   ↓
Spring MVC (DispatcherServlet) maps it to a controller method
   ↓
Spring Core manages objects and dependencies
   ↓
Spring Boot auto-configured all of the above
   ↓
Developer writes only business logic
```

---

## Interview Q&A
**Q1. What is the difference between Spring and Spring Boot?**
Spring is the framework that provides IoC/DI and modules (MVC, Data, Security). Spring Boot is built on Spring and removes setup work through auto-configuration, starters, and an embedded server. Boot uses Spring; it doesn't replace it.

**Q2. What is a Servlet Container? Name some.**
It's the runtime that handles low-level HTTP work (ports, parsing, threads) and calls servlets. Examples: Tomcat, Jetty, Undertow.

**Q3. Is Spring MVC a replacement for Servlets?**
No. Spring MVC is built on top of the Servlet API (its `DispatcherServlet` is itself a servlet). It gives a cleaner annotation-based programming model.

**Q4. PUT vs PATCH?**
PUT replaces the whole resource. PATCH updates part of it.

**Q5. What does "opinionated" mean for Spring Boot?**
It provides sensible defaults based on what is on the classpath, so you configure only what differs from the defaults.

**Q6. JPA vs Hibernate?**
JPA is a specification (interfaces and rules). Hibernate is one implementation of it.

**Q7. Are microservices a Spring module?**
No. Microservices are an architecture style. Spring Boot is commonly used to build them.

## Quick revision
- Client asks, server responds; HTTP is the format.
- Core Java sees raw bytes. Servlets and containers translate HTTP into Java calls.
- Spring = IoC + DI + modules. Spring Boot = auto-configured Spring.
- Spring MVC = Servlets + Spring Core.
- Data flow: Spring Data JPA → Hibernate → JDBC → Database.
