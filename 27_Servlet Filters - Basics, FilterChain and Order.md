# Servlet Filters: Basics, FilterChain and Order

> **Source:** Lecture 19 (`Notes_Of_Aditya/Lecture 19/code/filterDemo`)
> **Related:** [28_Advanced Filters](28_Advanced%20Filters%20-%20Wrappers%2C%20FilterRegistrationBean%20and%20OncePerRequestFilter.md) · [22_Spring MVC Architecture](22_Spring%20MVC%20Architecture%20and%20DispatcherServlet.md)

## Why this topic matters
Filters run **before Spring MVC** and are how logging, auth checks, trace IDs, CORS, and **Spring Security** work. Common questions: execution order, Filter vs Interceptor, and how to block a request.

---

## 1. Why filters? Cross-cutting concerns
Logging, auth-token checks, timing, request IDs, encoding, and security headers apply to **every** API. Writing them in every controller is repetitive, easy to forget, and clutters business code.
A **Filter** is one reusable HTTP-level checkpoint around every request.

## 2. Where filters sit
```
Client
  ↓
Tomcat
  ↓
Filter 1 → Filter 2 → … (servlet layer)       ← runs even for /error, static files, unknown URLs
  ↓
DispatcherServlet
  ↓
Interceptors (Spring MVC layer)
  ↓
Controller → Service → Repository
  ↑ response comes back through the same chain in REVERSE
```
Filters come from the **Servlet spec** (`jakarta.servlet.Filter`), not from Spring. Boot 2 used `javax.servlet`; Boot 3+ uses `jakarta.servlet`.
At filter time, Spring **hasn't chosen a controller yet**, so a filter knows the URL, method, headers, and body, but not "this is `StudentController.getStudent()`".

## 3. The Filter interface
```java
public interface Filter {
    default void init(FilterConfig cfg) throws ServletException {}           // once at startup
    void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) // every matching request
            throws IOException, ServletException;
    default void destroy() {}                                                  // once at shutdown
}
```
Pattern:
```java
before();                      // pre-processing
chain.doFilter(req, res);      // pass to next filter / DispatcherServlet
after();                       // post-processing (response on its way back)
```
**Two choices in every filter:**
- **Continue:** call `chain.doFilter(req, res)`.
- **Block:** set the status, write the body, and `return` **without** calling the chain. The controller never runs.

`ServletRequest`/`ServletResponse` are generic, so cast them to `HttpServletRequest`/`HttpServletResponse` for `getMethod()`, `getRequestURI()`, `getHeader()`, `setStatus()`, `setHeader()`…

## 4. Lecture code
### AuthenticationFilter (`@Order(1)`)
```java
@Component
@Order(1)
public class AuthenticationFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest  httpRequest  = (HttpServletRequest) req;
        HttpServletResponse httpResponse = (HttpServletResponse) res;

        String token  = httpRequest.getHeader("token");
        String apiKey = httpRequest.getHeader("x-api-key");

        if (token == null || !token.equals("12345")) {
            httpResponse.setStatus(HttpServletResponse.SC_UNAUTHORIZED);   // 401, empty body
            return;                                                        // ← request STOPS here
        }
        if (apiKey == null || !apiKey.equals("secret123")) {
            httpResponse.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            httpResponse.setContentType("application/json");
            httpResponse.getWriter().write("{ \"message\" : \"Invalid or missing Api key\" }");
            return;
        }
        chain.doFilter(req, res);                                          // ← continue
    }
}
```
### LoggingFilter (`@Order(2)`)
```java
@Component
@Order(2)
public class LoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        long start = System.currentTimeMillis();
        HttpServletRequest  httpRequest  = (HttpServletRequest) req;
        HttpServletResponse httpResponse = (HttpServletResponse) res;

        httpResponse.setHeader("X-Request-ID", UUID.randomUUID().toString()); // header BEFORE chain
        System.out.println("Incoming Request : " + httpRequest.getMethod() + " " + httpRequest.getRequestURI());
        try {
            chain.doFilter(req, res);
        } finally {                                                            // runs even if downstream throws
            System.out.println("Response status: " + httpResponse.getStatus());
            System.out.println("API Response time : " + (System.currentTimeMillis() - start));
        }
    }
}
```

### ✅ Verified by running the lecture app (`POST /api/students`)
| Request headers | Response | Console |
|---|---|---|
| none | `401`, **empty body** | *(nothing)* |
| `token: 12345` only | `401` + `{"message" : "Invalid or missing Api key"}` | *(nothing)* |
| `token: 12345` + `x-api-key: secret123` | `200 DONE` + header `X-Request-ID: <uuid>` | see below |

```
Incoming Request : POST /api/students
Student Created
Aditya
a@x.com
Response status: 200
API Response time : 56
```
> 💡 **Interview insight:** because `AuthenticationFilter` is `@Order(1)` and rejects early, **rejected requests are never logged** by `LoggingFilter` (order 2). If you need an audit log of failed logins, the logging filter must run **first**. Order is a design decision.

## 5. Execution order
```java
@Order(1) AuthFilter:    print("Auth before");    chain.doFilter(); print("Auth after");
@Order(2) LoggingFilter: print("Logging before"); chain.doFilter(); print("Logging after");
```
```
Auth before
Logging before
Controller
Logging after
Auth after
```
The request goes **in order**, the response comes back in **reverse** (like nested method calls or a stack).
Lower `@Order` runs earlier. Without `@Order`, the order isn't guaranteed, so set it explicitly when it matters.

## 6. Common use cases
| Use case | Snippet |
|---|---|
| Request logging | `getMethod()`, `getRequestURI()`, `getQueryString()`, `getRemoteAddr()`, `getHeader("User-Agent")` |
| Timing | `start`, then `try { chain } finally { duration }` |
| Trace/correlation ID | `setHeader("X-Request-ID", UUID…)`; also put it in MDC for logs |
| Common/security headers | `X-Content-Type-Options: nosniff`, etc. |
| Encoding | `setCharacterEncoding("UTF-8")` (Spring has `CharacterEncodingFilter`) |
| Early rejection | Missing API key → 401 + `return` |

## 7. Good practices
- **Add headers before `chain.doFilter()`.** After the body is written, the response may be **committed** and header changes are ignored.
- Use **`try/finally`** around the chain for timing and cleanup.
- Use **SLF4J** (`logger.info("method={}, path={}", ...)`), not `System.out`.
- **Never log** `Authorization` tokens, API keys, cookies, passwords, or card or personal data. Mask them.
- Real authentication belongs in **Spring Security** (itself a chain of filters), not homemade header checks.
- Hardcoded `"12345"` / `"secret123"` are for the demo only. Use config/secret storage, and compare secrets with `MessageDigest.isEqual` to avoid timing attacks.
- Exceptions thrown in a filter **don't reach `@RestControllerAdvice`** (DispatcherServlet hasn't run yet). Write the error response inside the filter.

## 8. Filter vs Interceptor vs AOP
| | Filter | HandlerInterceptor | AOP (`@Aspect`) |
|---|---|---|---|
| Spec | Servlet | Spring MVC | Spring AOP |
| Runs | Before/after DispatcherServlet | Before/after the controller method | Around any Spring bean method |
| Knows the controller method? | ❌ | ✅ (`HandlerMethod`) | ✅ |
| Applies to static files / `/error` | ✅ | Only MVC-handled requests | ❌ |
| Can replace request/response objects | ✅ (wrappers) | ❌ | ❌ |
| Typical use | Logging, auth, CORS, encoding, trace ID | Per-handler auth, locale, model attributes | Transactions, method logging, metrics |

---

## Interview Q&A
**Q1. What is a Filter? Where does it run?**
A Servlet-spec component that intercepts requests and responses in the servlet container, before DispatcherServlet and after the response comes back.

**Q2. How do you stop a request in a filter?**
Write the response (status/body) and return without calling `chain.doFilter()`.

**Q3. In what order do multiple filters run?**
Request: ascending `@Order` (or `FilterRegistrationBean.setOrder`). Response: reverse.

**Q4. Filter vs Interceptor?**
Filters are servlet-level, run for every matching request (including static and error), and can wrap request/response. Interceptors are Spring MVC-level, know the handler method, and have `preHandle`/`postHandle`/`afterCompletion`.

**Q5. Why use `try/finally` around `chain.doFilter`?**
So timing and cleanup run even if downstream code throws.

**Q6. Why can't `@ControllerAdvice` handle exceptions from filters?**
Filters run outside DispatcherServlet's exception-resolution mechanism.

**Q7. How is Spring Security related to filters?**
It's implemented as a `FilterChainProxy` (a `DelegatingFilterProxy` → `SecurityFilterChain`) containing many ordered filters (authentication, authorization, CSRF, …).

**Q8. Why add response headers before calling the chain?**
After downstream writes or flushes the body, the response is committed and headers can no longer change.

## Quick revision
- Filter = servlet-level checkpoint: before → `chain.doFilter` → after.
- No `chain.doFilter` + `return` = blocked.
- `@Order`: lower runs first; response runs in reverse. Order decides what gets logged.
- Headers before the chain, timing in `finally`, never log secrets, real auth goes in Spring Security.
