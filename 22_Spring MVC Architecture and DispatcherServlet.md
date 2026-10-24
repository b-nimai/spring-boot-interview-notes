# Spring MVC Architecture and DispatcherServlet

> **Source:** Lecture 15 (`Notes_Of_Aditya/Lecture 15/code/SpringMVCDemo`)
> **Related:** [21_Servlets](21_Servlets%2C%20Tomcat%20and%20Servlet%20Lifecycle.md) · [23_Spring MVC with JSP](23_Spring%20MVC%20with%20JSP%20and%20ViewResolver.md) · [27_Servlet Filters](27_Servlet%20Filters%20-%20Basics%2C%20FilterChain%20and%20Order.md)

## Why this topic matters
"Explain the flow of a request in Spring MVC" is one of the most frequently asked Spring interview questions. This lecture builds Spring MVC **without Spring Boot**, so you can see exactly what Boot auto-configures.

---

## 1. What is Spring MVC?
Spring's web framework (module **`spring-webmvc`**) for web apps and REST APIs, **built on the Servlet API**.
Spring Boot doesn't replace it. Boot just **configures it automatically**.

## 2. DispatcherServlet: the Front Controller
One central servlet receives **every** request and dispatches it to the right controller method.
It doesn't do everything itself; it **coordinates** helper components:

| Component | Job |
|---|---|
| `DispatcherServlet` | Front controller; receives every request and coordinates the rest |
| `HandlerMapping` (`RequestMappingHandlerMapping`) | Finds **which** controller method matches (path + HTTP method + headers…) |
| `HandlerAdapter` (`RequestMappingHandlerAdapter`) | **Invokes** that method |
| `HandlerMethodArgumentResolver` | Fills parameters: `@PathVariable`, `@RequestParam`, `@RequestBody`, `@RequestHeader`… |
| `HttpMessageConverter` (+ **Jackson**) | JSON ↔ Java object |
| `HandlerMethodReturnValueHandler` | Handles return values (`ResponseEntity`, `@ResponseBody`, view names) |
| `HandlerExceptionResolver` | Turns exceptions into responses (`@ExceptionHandler`, `@ControllerAdvice`) |
| `ViewResolver` | View name → JSP/Thymeleaf page (for `@Controller`) |

## 3. Request flow (memorize this)
```
Client
  ↓ HTTP
Tomcat (connector thread)
  ↓
Filters (servlet layer)
  ↓
DispatcherServlet
  ↓ 1. HandlerMapping:  "GET /students/1" → StudentController.getStudent()
  ↓ 2. HandlerAdapter:  resolve args (@PathVariable id = 1) → invoke method
  ↓        (Interceptors preHandle / postHandle run around this)
Controller → Service → Repository
  ↓ 3. Return value → HttpMessageConverter (Jackson) → JSON
  ↓    (or view name → ViewResolver → JSP)
DispatcherServlet → Filters → Tomcat → Client
```
**When is the mapping table built?** At startup: component scan finds `@Controller` beans, `RequestMappingHandlerMapping` reads `@RequestMapping`/`@GetMapping`… and builds:
```
GET    /students/{id} → StudentController.getStudent()
POST   /students      → StudentController.createStudent()
GET    /students      → StudentController.getAllStudent()
```
Same path, different HTTP method = different handler.

## 4. Where request data comes from
| Source | Example | Annotation |
|---|---|---|
| Path | `GET /students/10` | `@PathVariable Long id` |
| Query string | `GET /students?name=Rohit` | `@RequestParam String name` |
| JSON body | `POST /students` + `{...}` | `@RequestBody Student s` (Jackson; needs a no-arg constructor + setters) |

## 5. Manual Spring MVC + embedded Tomcat (no Boot)
### pom.xml
```xml
<dependency><groupId>org.springframework</groupId><artifactId>spring-webmvc</artifactId><version>7.0.7</version></dependency>
<dependency><groupId>org.apache.tomcat.embed</groupId><artifactId>tomcat-embed-core</artifactId><version>11.0.22</version></dependency>
<dependency><groupId>com.fasterxml.jackson.core</groupId><artifactId>jackson-databind</artifactId><version>2.21.4</version></dependency>
```
### WebConfig
```java
@Configuration
@ComponentScan(basePackages = "in.strikes")
@EnableWebMvc                    // registers HandlerMapping, HandlerAdapter, message converters…
public class WebConfig { }
```
### Main: what Spring Boot does for you
```java
public static void main(String[] args) throws LifecycleException {
    Tomcat tomcat = new Tomcat();                                  // 1. embedded server
    tomcat.setPort(8080);
    tomcat.getConnector();                                         // 2. HTTP connector

    Context context = tomcat.addContext("",                        // 3. context path "" → /students (not /app/students)
            new File("src/main/webapp").getAbsolutePath());        //    docBase folder

    AnnotationConfigWebApplicationContext springContext =
            new AnnotationConfigWebApplicationContext();           // 4. Spring IoC container (web-aware)
    springContext.register(WebConfig.class);

    DispatcherServlet dispatcherServlet = new DispatcherServlet(springContext); // 5. front controller gets the container
    Tomcat.addServlet(context, "dispatcherServlet", dispatcherServlet);         // 6. register with Tomcat
    context.addServletMappingDecoded("/", "dispatcherServlet");                 // 7. ALL requests → DispatcherServlet

    tomcat.start();
    tomcat.getServer().await();                                    // 8. keep main() alive
}
```
**Why does DispatcherServlet need the Spring context?** It must find controllers, mappings, converters, and services, and all of those live in the IoC container.

### Controller / Service / Repository (in-memory)
```java
@RestController
@RequestMapping("/students")
public class StudentController {
    @PostMapping
    public ResponseEntity<Student> createStudent(@RequestBody Student s) { return ResponseEntity.ok(studentService.createStudent(s)); }

    @GetMapping("/{id}")
    public ResponseEntity<Student> getStudent(@PathVariable("id") Long id) {
        Student s = studentService.getStudent(id);
        return s == null ? ResponseEntity.notFound().build() : ResponseEntity.ok(s);
    }

    @GetMapping
    public ResponseEntity<List<Student>> getAllStudent() { return ResponseEntity.ok(studentService.getAllStudent()); }
}

@Repository
public class StudentRepository {
    private Map<Long, Student> studentDB = new HashMap<>();    // fake DB (not thread-safe: use ConcurrentHashMap)
    public Student save(Student s) { studentDB.put(s.getId(), s); return s; }
    public Student findById(Long id) { return studentDB.get(id); }
    public List<Student> findAll() { return new ArrayList<>(studentDB.values()); }
}
```
> The lecture adds `@ResponseBody` on each method. That's redundant with `@RestController` (which already includes it), but harmless.

## 6. Manual Spring MVC vs Spring Boot
| Task | Manual | Spring Boot |
|---|---|---|
| Spring MVC / Tomcat / Jackson deps | Add each one | One starter (`spring-boot-starter-web`, or `-webmvc` in Boot 4) |
| Start Tomcat | `new Tomcat()…start()` | Automatic |
| Create web ApplicationContext | Manual | Automatic |
| Register and map DispatcherServlet | Manual | Automatic (`DispatcherServletAutoConfiguration`) |
| Enable MVC | `@EnableWebMvc` | `WebMvcAutoConfiguration` |
| Component scan | `@ComponentScan` | `@SpringBootApplication` |

> ⚠️ In Spring Boot, **don't add `@EnableWebMvc`**. It switches off Boot's MVC auto-configuration (static resources, message-converter customizations, etc.). Implement `WebMvcConfigurer` instead.

## 7. `@Controller` vs `@RestController`
| `@Controller` | `@RestController` |
|---|---|
| Return value = **view name** → ViewResolver → HTML page | Return value = **response body** (JSON via Jackson) |
| Needs `@ResponseBody` per method to return data | = `@Controller` + `@ResponseBody` |

---

## Interview Q&A
**Q1. Explain the request flow in Spring MVC.**
Tomcat → filters → DispatcherServlet → HandlerMapping finds the handler → HandlerAdapter resolves arguments and invokes it (interceptors around) → the return value goes through a message converter (JSON) or a ViewResolver (view) → response. Exceptions go to HandlerExceptionResolvers (`@ControllerAdvice`).

**Q2. What is DispatcherServlet? What design pattern does it use?**
The central `HttpServlet` of Spring MVC, implementing the **Front Controller** pattern.

**Q3. HandlerMapping vs HandlerAdapter?**
HandlerMapping *finds* the handler. HandlerAdapter *executes* it, hiding the handler type from DispatcherServlet.

**Q4. Who converts JSON to objects?**
`HttpMessageConverter`, chosen by `Content-Type`/`Accept`. With Jackson 2 (Spring 6 / Boot 3) it's `MappingJackson2HttpMessageConverter`. Spring 7 / Boot 4 default to Jackson 3 (`JacksonJsonHttpMessageConverter`, package `tools.jackson`). Jackson 2 still works but is deprecated.

**Q5. What does `@EnableWebMvc` do? Should you use it with Spring Boot?**
It imports Spring MVC's default infrastructure config. In Boot, it disables MVC auto-configuration, so avoid it unless you want full manual control.

**Q6. Is Spring MVC built on servlets?**
Yes. DispatcherServlet extends `HttpServlet` and runs inside a servlet container.

**Q7. What does Spring Boot auto-configure for MVC?**
Embedded Tomcat, DispatcherServlet registration at `/`, MVC infrastructure, Jackson converters, error handling (`/error`), static resources.

## Quick revision
- DispatcherServlet = front controller.
- HandlerMapping finds the method, HandlerAdapter calls it, ArgumentResolvers fill parameters, MessageConverter produces JSON.
- Manual setup: Tomcat + web context + DispatcherServlet mapped to `/` + `@EnableWebMvc`.
- Boot does all of that. Don't add `@EnableWebMvc` in Boot.
