# Spring MVC with JSP and ViewResolver

> **Source:** Lecture 15 (`Notes_Of_Aditya/Lecture 15/code/JSPDemo`)
> **Related:** [22_Spring MVC Architecture](22_Spring%20MVC%20Architecture%20and%20DispatcherServlet.md)

## Why this topic matters
Spring MVC was originally a **server-side rendered** framework (Model-View-Controller). Even if you only build REST APIs, interviewers ask what MVC means, what a ViewResolver does, and `@Controller` vs `@RestController`.

---

## 1. MVC meaning
| Part | Here |
|---|---|
| **Model** | Data passed from controller to view (`Model` / `model.addAttribute`) |
| **View** | JSP page that renders HTML (`home.jsp`) |
| **Controller** | Handles the request, fills the model, returns a **view name** |

```
GET /  → DispatcherServlet → HandlerMapping → HelloController.showHomePage()
      → controller adds "message" to Model, returns "home"
      → ViewResolver: "home" → /WEB-INF/views/home.jsp
      → JSP rendered with ${message} → HTML response
```

## 2. Extra dependency: Jasper (JSP engine)
```xml
<dependency>
    <groupId>org.apache.tomcat.embed</groupId>
    <artifactId>tomcat-embed-jasper</artifactId>
    <version>11.0.22</version>
</dependency>
```
JSPs are compiled into servlets by Jasper. In the repo you can see the generated `tomcat.8080/work/.../home_jsp.java`, which shows that **a JSP becomes a servlet**.

## 3. WebConfig with a ViewResolver
```java
@Configuration
@ComponentScan(basePackages = "in.strikes")
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");      // "home" → /WEB-INF/views/home.jsp
        resolver.setSuffix(".jsp");
        return resolver;
    }

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/assets/**")          // URL pattern
                .addResourceLocations("/assets/");         // folder under src/main/webapp
    }
}
```
- **ViewResolver** maps a *logical view name* to an *actual view file*.
- **Resource handler:** because DispatcherServlet is mapped to `/`, it receives `/assets/style.css` too. Without a handler, CSS returns 404.

## 4. Main: registering the JSP servlet too
```java
Context context = tomcat.addContext("", new File("src/main/webapp").getAbsolutePath()); // docBase matters now
context.addServletContainerInitializer(new JasperInitializer(), Set.of());

Wrapper jspWrapper = Tomcat.addServlet(context, "jsp", new JspServlet());   // compiles/serves *.jsp
jspWrapper.setLoadOnStartup(3);
context.addServletMappingDecoded("*.jsp", "jsp");

DispatcherServlet ds = new DispatcherServlet(springContext);
Wrapper dw = Tomcat.addServlet(context, "dispatcherServlet", ds);
dw.setLoadOnStartup(1);                                                      // init at startup, not first request
context.addServletMappingDecoded("/", "dispatcherServlet");
```
`setLoadOnStartup(n)`: initialize the servlet at deployment. Lower numbers start first.

## 5. Controller: returns view names
```java
@Controller                                     // NOT @RestController
public class HelloController {

    @GetMapping("/")
    public String showHomePage(Model model) {
        model.addAttribute("message", "Enter your name");
        return "home";                          // view name, NOT the response body "home"
    }

    @PostMapping("/greet")
    public String greetUser(@RequestParam("name") String name, Model model) {
        model.addAttribute("message", "Hello " + name);   // form field name="name"
        return "home";
    }
}
```

## 6. The view: `src/main/webapp/WEB-INF/views/home.jsp`
```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head><link rel="stylesheet" href="/assets/style.css"></head>
<body>
  <h1>Spring MVC JSP Demo</h1>
  <p>${message}</p>                               <!-- EL: reads Model attribute -->
  <form action="/greet" method="post">
      <input type="text" name="name" required>
      <button type="submit">Submit</button>
  </form>
</body>
</html>
```
**Why `WEB-INF`?** Files under `WEB-INF` **can't be requested directly** by the browser. Every page must go through a controller, so the model is always prepared.

Flow: open `/`, which shows "Enter your name". Submit "Aditya", which POSTs to `/greet` and renders "Hello Aditya".

## 7. REST vs server-rendered
| REST API | JSP / server-side MVC |
|---|---|
| `@RestController` | `@Controller` |
| Returns objects → JSON | Returns view name → HTML |
| HttpMessageConverter | ViewResolver |
| Frontend (React/mobile) renders the UI | Server renders the UI |

In Spring Boot, JSP is discouraged (limitations with executable JARs). **Thymeleaf** is the usual template engine: add `spring-boot-starter-thymeleaf` and put templates in `src/main/resources/templates/`. Boot auto-configures its ViewResolver.

> Pitfall (POST-redirect-GET): returning a view directly from a POST means a browser refresh re-submits the form. Real apps `return "redirect:/..."`.

---

## Interview Q&A
**Q1. What is a ViewResolver?**
It maps the logical view name returned by a controller to an actual `View` (e.g. `InternalResourceViewResolver` adds a prefix/suffix to locate a JSP).

**Q2. `@Controller` vs `@RestController`?**
`@Controller` returns view names (unless `@ResponseBody` is used). `@RestController` = `@Controller` + `@ResponseBody`, so return values are serialized into the body.

**Q3. What is the `Model` object?**
A map of attributes the controller passes to the view for rendering.

**Q4. Why keep JSPs inside `WEB-INF`?**
So they aren't directly accessible; requests must go through controllers.

**Q5. Why did CSS return 404?**
DispatcherServlet mapped to `/` intercepts static paths. Register resource handlers (`addResourceHandlers`). Boot does this automatically for `static/`.

**Q6. What does `redirect:` do in a view name?**
It sends a 302 redirect instead of rendering, which is used for the Post/Redirect/Get pattern.

## Quick revision
- MVC = Model (data) + View (JSP) + Controller (returns view name).
- `InternalResourceViewResolver` prefix + suffix → JSP path.
- `@Controller` + `Model` + `${attr}` in the JSP.
- JSPs live in `WEB-INF`; static files need resource handlers.
