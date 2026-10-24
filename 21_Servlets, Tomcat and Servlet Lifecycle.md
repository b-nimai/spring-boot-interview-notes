# Servlets, Tomcat and the Servlet Lifecycle

> **Source:** Lecture 14 (`Notes_Of_Aditya/Lecture 14/code/ServletCrudDemo`)
> **Related:** [01_Introduction](01_Introduction%20to%20Spring%20and%20Spring%20Boot.md) · [22_Spring MVC Architecture](22_Spring%20MVC%20Architecture%20and%20DispatcherServlet.md)

## Why this topic matters
Spring MVC **is** a servlet (`DispatcherServlet`), and Spring Boot just embeds Tomcat. Knowing servlets explains filters, the request lifecycle, thread safety, and "who actually listens on port 8080".

---

## 1. The problem servlets solve
A browser speaks **HTTP text**, not Java:
```http
GET /hello HTTP/1.1
Host: localhost:8080
```
With raw `ServerSocket` you'd have to parse HTTP, route URLs, build responses, manage threads, and keep the server alive yourself.
**Servlet idea:** let a **container** handle HTTP and call **your Java class** with ready-made objects.

```
Browser ──HTTP──► Tomcat (listens on 8080)
                    │  parses HTTP, creates HttpServletRequest / HttpServletResponse
                    ▼
               Your Servlet.doGet(req, resp)  ← business logic
                    │  writes response
                    ▼
                 Tomcat ──HTTP──► Browser
```
> ❌ "My servlet listens on port 8080."
> ✅ **Tomcat listens.** The servlet lives inside Tomcat and is *called* by Tomcat.

**Static vs dynamic:** `GET /index.html` returns a file (no Java needed). `GET /user?id=101` needs logic, which is where servlets come in.

## 2. Tomcat = Servlet Container (and IoC again)
Tomcat creates the servlet object, calls `init()`/`service()`/`doXxx()`/`destroy()`, creates the request and response objects, manages the **thread pool**, maps URLs, and sends responses.

| World | Container | Manages |
|---|---|---|
| Servlet | Tomcat | Servlets |
| Spring | ApplicationContext | Beans |

In both, **you don't call `new` or the methods; the container does.** That's Inversion of Control.

## 3. First servlet (lecture code)
```java
@WebServlet("/hello")                         // URL mapping (replaces web.xml)
public class HelloServlet extends HttpServlet {

    public HelloServlet() { System.out.println("HelloServlet Constructor called"); }

    @Override
    public void init() { System.out.println("init() method called"); }

    @Override
    public void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        response.setStatus(200);
        response.setContentType("text/plain");
        response.getWriter().write("Hello");
    }

    @Override
    public void destroy() { System.out.println("Destroy method called"); }
}
```
Console, deploying and hitting `/hello` twice, then stopping Tomcat:
```
HelloServlet Constructor called     ← first request only (lazy by default)
init() method called                ← once
                                    ← 2nd request: only service() → doGet()
Destroy method called               ← on shutdown/undeploy
```

## 4. Servlet lifecycle
```
Load class → constructor → init()          (ONCE, on first request or at startup if load-on-startup ≥ 0)
               ↓
          service(req, resp)               (EVERY request, each on its own thread)
               ↓ dispatches by HTTP method
     doGet / doPost / doPut / doDelete
               ↓
          destroy()                        (ONCE, on shutdown/undeploy/reload)
```
- `HttpServlet.service()` already routes to `doGet`/`doPost`/…, so **override the `doXxx` methods, not `service()`**.
- An unimplemented method returns **405 Method Not Allowed** by default.
- `init()`: one-time setup (load config, create expensive resources). `destroy()`: release resources.

### ⚠️ One instance, many threads
Tomcat normally creates **one servlet object** and calls it from **many threads at once**. Instance fields are shared:
```java
private UserService userService = new UserService();   // shared by every request
// UserService uses a HashMap → NOT thread-safe under concurrent POSTs
```
Use `ConcurrentHashMap`, keep servlets stateless, or synchronize. The same rule applies to Spring singleton controllers.

## 5. CRUD servlet (lecture code, condensed)
```java
@WebServlet("/users")
public class UserServlet extends HttpServlet {
    private UserService userService = new UserService();          // in-memory Map<Integer, User>

    @Override
    public void doPost(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        Integer id   = Integer.parseInt(req.getParameter("id"));   // POST /users?id=1&name=..&email=..&mobile=..
        String name  = req.getParameter("name");
        ...
        userService.createUser(new User(id, name, email, mobile));
        resp.setStatus(201);
        resp.setContentType("application/json");
        resp.getWriter().write("{ \"message\" : \"User Added successfully\" }");
    }

    @Override
    public void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        String idParam = req.getParameter("id");
        if (idParam == null) {                                     // GET /users → all
            resp.getWriter().write(usersToJson(userService.getAllUsers()));
            return;
        }
        User u = userService.getUserById(Integer.parseInt(idParam)); // GET /users?id=1
        ...
        resp.getWriter().write(userToJson(u));                     // JSON built BY HAND
    }
}
```
This shows how much plumbing Spring MVC removes: manual parameter parsing, manual JSON, manual status codes, one class per URL.

### ⚠️ Bugs in the lecture servlet (good "spot the bug" practice)
1. `Integer.parseInt(request.getParameter("id"))` runs **before** the null check. A missing `id` throws `NumberFormatException` (500), so `id == null` can never be true.
2. After writing the **400** response there's **no `return`**, so execution continues, creates the user, and writes a second body.
3. In `doGet`, after setting **404** there's **no `return`**. `userToJson(null)` then throws `NullPointerException`, and the status is overwritten with 200.
4. `userToJson` doesn't quote string values (`"name" : Rahul`), so it's **invalid JSON**. Use Jackson.
5. Query parameters are used for POST data to keep the demo simple. Real APIs send a JSON body (`req.getReader()` + Jackson).

## 6. External Tomcat and WAR deployment
```xml
<packaging>war</packaging>

<dependency>
    <groupId>jakarta.servlet</groupId>
    <artifactId>jakarta.servlet-api</artifactId>
    <version>6.1.0</version>
    <scope>provided</scope>      <!-- compile against it, DON'T package it -->
</dependency>
```
- **WAR** (Web ARchive) = classes + servlets + static files + `WEB-INF/web.xml` + libs.
- **`provided` scope:** needed to compile, but Tomcat's own `lib/` provides the Servlet API at runtime. Packaging a second copy can cause classloader conflicts.
- Deploy: copy `target/app.war` to `tomcat/webapps/`. Tomcat explodes it, and the URL becomes `http://localhost:8080/app/...` (**context path** = WAR name).
- One external Tomcat can host **many** apps (`/app1`, `/app2`).

```bash
cd apache-tomcat-10.1.x && chmod +x bin/*.sh
./bin/catalina.sh run         # foreground (Ctrl+C to stop)
./bin/startup.sh / shutdown.sh
lsof -i :8080 ; kill -9 <pid> # port busy
```
> `javax.servlet` (Tomcat ≤ 9, Boot 2) → **`jakarta.servlet`** (Tomcat 10+, Boot 3+).

## 7. External vs embedded Tomcat
| External | Embedded (Spring Boot) |
|---|---|
| Install Tomcat, build WAR, deploy into `webapps` | `java -jar app.jar`; Tomcat starts **inside** the app |
| "Application goes inside the server" | "Server starts inside the application" |
| Many apps per server | One app per process (fits containers and microservices) |

Spring Boot didn't remove servlets. `spring-boot-starter-web(mvc)` embeds Tomcat and registers **one** servlet, `DispatcherServlet`, mapped to `/`.

## 8. From servlets to Spring MVC
```
Servlet style:   /users → UserServlet   /products → ProductServlet   /orders → OrderServlet
Spring MVC:      /users, /products, /orders ──► DispatcherServlet ──► @Controller methods
```
This is the **Front Controller pattern**. See [22_Spring MVC Architecture](22_Spring%20MVC%20Architecture%20and%20DispatcherServlet.md).

---

## Interview Q&A
**Q1. What is a Servlet? What is a Servlet Container?**
A servlet is a Java class (`HttpServlet`) that handles HTTP requests. The container (Tomcat, Jetty, Undertow) manages servlet lifecycles, threads, and HTTP parsing, and calls servlets.

**Q2. Explain the servlet lifecycle.**
Load → instantiate → `init()` once → `service()` per request (dispatching to `doGet`/`doPost`…) → `destroy()` once at shutdown.

**Q3. Is a servlet thread-safe? How many instances exist?**
Usually one instance per servlet declaration, shared by many request threads. It's **not** thread-safe by default, so avoid mutable instance state.

**Q4. When is a servlet instantiated?**
On the first matching request by default. With `loadOnStartup` ≥ 0 (or `<load-on-startup>`), at deployment.

**Q5. Why `provided` scope for `jakarta.servlet-api`?**
The container supplies it at runtime. Bundling it can cause duplicate-class conflicts.

**Q6. JAR vs WAR?**
A WAR is a web-app archive deployed into an external container. Spring Boot's executable JAR embeds the container.

**Q7. What does `service()` do in `HttpServlet`?**
It reads the HTTP method and delegates to the matching `doXxx`. Unimplemented methods return 405.

**Q8. How is Spring MVC related to servlets?**
`DispatcherServlet` is an `HttpServlet` acting as a front controller. Spring MVC is built on top of the Servlet API.

## Quick revision
- Tomcat listens, parses HTTP, and calls your servlet.
- Lifecycle: constructor + `init` (once) → `service`/`doXxx` (per request, multi-threaded) → `destroy` (once).
- One servlet instance means shared state, so be thread-safe.
- WAR + external Tomcat (`provided` API) vs Boot's embedded Tomcat.
- Spring MVC = one `DispatcherServlet` instead of many servlets.
