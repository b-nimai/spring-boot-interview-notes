# First Spring Boot Application

> **Source:** Lecture 02 (`Notes_Of_Aditya/Lecture 02/code/demo`)

## Why this topic matters
This is your first working web app. It shows how fast Spring Boot is. Every line of it hides something (Tomcat, Spring MVC, component scanning) that you will be asked about later.

---

## 1. Port, IP and localhost
- **IP address** identifies the **machine** (`192.168.1.10`).
- **Port** identifies the **application** on that machine (`:8080`).
- `localhost` = your own machine = `127.0.0.1`.
- `http://localhost:8080/hello` = "my own machine, the app listening on port 8080, path `/hello`".

Default ports: **HTTP → 80**, **HTTPS → 443**. DNS gives the IP. The port comes either from the URL or from the protocol default.

```
192.168.1.10
 ├── Chrome          :5000
 ├── WhatsApp        :5222
 └── Spring Boot App :8080
```

## 2. Spring Initializr
Spring Initializr (start.spring.io or IntelliJ) is a **project generator**. It creates the folder structure, `pom.xml`, Boot version, Java version, and dependencies.
- **Dependency** = an external library your project needs (Spring Web, MySQL connector, Lombok…).
- **Versions:** `SNAPSHOT` = work in progress, `RC` = release candidate, **Stable** = use this one.
- **Packaging:** modern Boot apps use a **JAR** (an executable "fat" JAR with an embedded server), so there's no external server to deploy to.

## 3. Project structure
```
demo/
├── pom.xml                         ← Maven config + dependencies
├── src/main/java/.../DemoApplication.java   ← entry point (main)
├── src/main/java/.../HelloController.java
├── src/main/resources/application.properties ← app config (port etc.)
└── src/test/java/...               ← tests
```

## 4. The code

### Main class
```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args); // starts context + embedded Tomcat
    }
}
```

### Controller
```java
@RestController                        // this class handles HTTP requests and returns data directly
public class HelloController {

    @GetMapping("hello")               // GET /hello → this method
    public String hello() {
        return "<h1>Hello World</h1>"; // response body sent to the browser
    }

    @GetMapping("bye")
    public String greetBye() {
        return "<h1>Bye</h1>";
    }
}
```
- **Controller** = the "receptionist" of the app. It receives a request and decides the response.
- `@RestController` = the class can handle HTTP requests and returns data (not a view name).
- `@GetMapping("/hello")` = maps a **GET** request on `/hello` to this method.

### Changing the port (`application.properties`)
```properties
spring.application.name=demo
server.port=9090
```
The app is now served at `http://localhost:9090/hello`. Spring Boot reads the property and **embedded Tomcat** listens on that port.

## 5. What happened when we ran it?
Console shows: `Tomcat started on port 8080`.
- We did **not** install Tomcat or write socket code.
- `spring-boot-starter-web` brought an **embedded Tomcat**, and Boot started and configured it.

```
Browser → localhost:8080/hello
   ↓
Embedded Tomcat receives the request
   ↓
Spring MVC finds the mapping /hello → hello()
   ↓
hello() returns "Hello World"
   ↓
Response goes back to the browser
```

What Boot did for us behind those few lines:
- Created the ApplicationContext
- Started embedded Tomcat
- Configured Spring MVC
- Detected the controller (component scan)
- Registered the `/hello` endpoint
- Handled the request and response

## 6. Why learn internals anyway?
"Spring Boot gives speed. Spring fundamentals give control."
Example: you get a **404** for your endpoint. To fix it you need to ask:
- Did component scanning find the controller? Is it in a sub-package of the main class?
- Was the bean created? Was the mapping registered? Is the URL or port right?
- Is there a context path, or is security blocking the request?

---

## Common pitfalls
- Controller placed **outside** the main class's package means it isn't scanned, so you get a 404.
- Port already in use: `Port 8080 was already in use`. Change `server.port` or stop the other process.
- Forgetting `spring-boot-starter-web` means no Tomcat starts and the app exits immediately.

## Interview Q&A
**Q1. What does `SpringApplication.run()` do?**
It creates the ApplicationContext, runs component scanning and auto-configuration, creates beans, starts the embedded server (if web), and runs any runners.

**Q2. `@RestController` vs `@Controller`?**
`@RestController` = `@Controller` + `@ResponseBody`. Return values are written directly to the HTTP response body (e.g. JSON or a string), not resolved as view names.

**Q3. How do you change the default port?**
Set `server.port=9090` in `application.properties` (or pass `--server.port=9090` on the command line).

**Q4. Who actually listens on port 8080?**
The embedded servlet container (Tomcat by default). Spring Boot configures and starts it.

**Q5. Why is a Spring Boot JAR runnable with `java -jar`?**
It's an executable "fat" JAR that bundles your code, dependencies, and the embedded server.

**Q6. What is localhost?**
The loopback address of your own machine (`127.0.0.1`).

## Quick revision
- IP → machine, port → application, localhost → my machine.
- `@SpringBootApplication` + `SpringApplication.run()` boots everything.
- `@RestController` + `@GetMapping` give you an endpoint.
- `server.port` changes the port. Embedded Tomcat is the real server.
