# Spring MVC Interceptors

> **Source:** Lecture 21 (`Notes_Of_Aditya/Lecture 21/code/interceptorDemo`)
> **Related:** [27_Servlet Filters](27_Servlet%20Filters%20-%20Basics%2C%20FilterChain%20and%20Order.md) · [22_Spring MVC Architecture](22_Spring%20MVC%20Architecture%20and%20DispatcherServlet.md)

## Why this topic matters
Interceptors run **inside Spring MVC, around the selected controller method**. Unlike filters, they know *which* controller and method will run. "Filter vs Interceptor" and "interceptor lifecycle order" are frequent interview questions.

---

## 1. Where an interceptor sits
```
Client → Tomcat → Filters → DispatcherServlet
                              ↓ HandlerMapping finds handler → HandlerExecutionChain (handler + interceptors)
                              ↓ Interceptor.preHandle()
                              ↓ HandlerAdapter → Controller
                              ↓ Interceptor.postHandle()
                              ↓ view rendering / response processing
                              ↓ Interceptor.afterCompletion()
                            ← Filters ← Client
```
Why the name *Handler*Interceptor? The handler isn't always a controller method. It can also be a static-resource handler. For controllers, it's a `HandlerMethod`.

## 2. The three callbacks
| Method | Runs | Typical use |
|---|---|---|
| `boolean preHandle(req, res, handler)` | **Before** the controller | Auth checks, logging, start timer, set request attributes. `return false` = **stop** |
| `void postHandle(req, res, handler, ModelAndView)` | After the controller **succeeds**, before view rendering | Add common model attributes (MVC views). **Not called if the controller throws** |
| `void afterCompletion(req, res, handler, Exception ex)` | After the whole request completes (like `finally`) | Final logging, timing, cleanup |

All three are `default` methods, so override only what you need.

> ⚠️ For `@RestController`/`ResponseEntity`, the body is usually **already written** before `postHandle`, and `modelAndView == null`. Don't try to change REST bodies there. Use `ResponseBodyAdvice`.
> ⚠️ `ex == null` in `afterCompletion` doesn't prove success: `@RestControllerAdvice` may already have handled the exception. Check `response.getStatus()`.

## 3. Lecture code
### Interceptors
```java
@Component
public class LoggingInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        System.out.println("HTTP Method: " + req.getMethod());
        System.out.println("Request URI: " + req.getRequestURI());
        System.out.println("Token Header: " + req.getHeader("token"));
        if (handler instanceof HandlerMethod hm) {                     // safe pattern-matching cast
            System.out.println("Controller: " + hm.getBeanType().getName());
            System.out.println("Controller Method: " + hm.getMethod().getName());
        }
        return true;
    }
    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse res, Object handler, Exception ex) {
        System.out.println("Response status: " + res.getStatus());
    }
}

@Component
public class AuthenticationInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        String apiKey = req.getHeader("x-api-key");
        if (apiKey != null && !apiKey.equals("secret123")) return false;   // ⚠️ see bugs below
        return true;
    }
}

@Component
public class AuthorizationInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) throws IOException {
        String role = req.getHeader("x-user-role");
        if (role != null && !role.equals("ADMIN")) {
            res.setStatus(HttpServletResponse.SC_FORBIDDEN);             // 403
            res.setContentType("application/json");
            res.getWriter().write("{ \"message\" : \"You are not Authorized to perform this action\" }");
            return false;
        }
        return true;
    }
}
```
### Registration (`WebMvcConfigurer`)
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    // constructor-injected interceptors …
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authenticationInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/auth/login", "api/public/**")   // ⚠️ missing leading "/"
                .order(1);
        registry.addInterceptor(loggingInterceptor).order(3);
        registry.addInterceptor(authorizationInterceptor).order(2);
    }
}
```
`@Component` only creates the bean. **`addInterceptors()` puts it in the MVC chain.** Forgetting this is the #1 mistake.

### ✅ Verified by running the lecture app (`POST /api/students`)
| Headers | Result | Expected (secure) |
|---|---|---|
| none | **200 "Student Created"** | 401 |
| `x-api-key: bad` | **200 with empty body** | 401 + message |
| `x-api-key: secret123`, `x-user-role: USER` | 403 + JSON message ✅ | 403 |
| `x-api-key: secret123`, `x-user-role: ADMIN` | 200 "Student Created" ✅ | 200 |

### ⚠️ Bugs this exposes (spot-the-bug practice)
1. **`apiKey != null && …` means a missing key passes.** The check is fail-open. Write it fail-closed:
   ```java
   if (!"secret123".equals(apiKey)) { res.setStatus(401); /* write body */ return false; }
   ```
   The same applies to the role check (no role header = allowed).
2. **Returning `false` without writing a response** gives the client an empty **200**. Always set the status and body before `return false`.
3. `"api/public/**"` lacks the leading `/`, so the exclusion is easy to get wrong. Write `"/api/public/**"`.
4. Because auth (order 1) runs before logging (order 3), **rejected requests are never logged** (same lesson as filters).
5. Real authentication and authorization belong in **Spring Security**.

## 4. Multiple interceptors: order
```
A.preHandle → B.preHandle → Controller → B.postHandle → A.postHandle → (render) → B.afterCompletion → A.afterCompletion
```
`preHandle` runs forward. `postHandle` and `afterCompletion` run in **reverse**. Lower `.order()` runs first.
If `B.preHandle` returns false, the controller and all `postHandle`s are skipped, and **only `A.afterCompletion`** runs (only interceptors whose `preHandle` returned true get `afterCompletion`).

## 5. Path patterns
| Pattern | Matches |
|---|---|
| `/api/students` | exact |
| `/api/students/*` | one segment below |
| `/api/students/**` | any depth below |
| `/**` | everything handled by MVC |
Combine `.addPathPatterns(...)` with `.excludePathPatterns(...)`.

## 6. Timing: don't use instance fields
```java
private long startTime;            // ❌ singleton bean shared across threads → wrong durations
```
```java
public boolean preHandle(HttpServletRequest req, ...) {
    req.setAttribute("start", System.nanoTime());          // ✅ per-request storage
    return true;
}
public void afterCompletion(HttpServletRequest req, HttpServletResponse res, Object h, Exception ex) {
    Long start = (Long) req.getAttribute("start");
    if (start == null) return;
    long ms = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);
    System.out.println(req.getMethod() + " " + req.getRequestURI() + " → " + res.getStatus() + " in " + ms + " ms");
}
```
Use `System.nanoTime()` for elapsed time. `currentTimeMillis()` is wall-clock and can jump.

## 7. 401 vs 403
| | Meaning |
|---|---|
| **401 Unauthorized** | Not authenticated: missing, invalid, or expired credentials |
| **403 Forbidden** | Authenticated but not allowed (e.g. not ADMIN) |

## 8. Filter vs Interceptor
| | Filter | Interceptor |
|---|---|---|
| Layer | Servlet | Spring MVC |
| Knows controller method | ❌ | ✅ (`HandlerMethod`, can read method annotations) |
| Before / after | Code around `chain.doFilter` | `preHandle` / `postHandle` + `afterCompletion` |
| Stop request | Don't call the chain | `return false` |
| Wrap request/response | ✅ | ❌ |
| Runs for static, `/error`, non-MVC servlets | ✅ | Only MVC-handled requests |
| Registered via | `@Component` / `FilterRegistrationBean` | `WebMvcConfigurer.addInterceptors` |
| Use for | Body caching, encoding, CORS, security chain | Handler-aware logging, custom-annotation checks, tenant context |

Combined order: `Filter1 → Filter2 → DispatcherServlet → I1.pre → I2.pre → Controller → I2.post → I1.post → I2.after → I1.after → Filter2 → Filter1`.

---

## Interview Q&A
**Q1. What is a HandlerInterceptor? Name its methods.**
A Spring MVC hook around handler execution: `preHandle` (before; can stop), `postHandle` (after success, before view), `afterCompletion` (always at the end).

**Q2. What happens when `preHandle` returns false?**
The chain stops: no controller, no `postHandle`. Earlier interceptors still get `afterCompletion`. You must write the response yourself.

**Q3. Is `postHandle` called if the controller throws?**
No. `afterCompletion` is (with `ex` set, unless an advice resolved it).

**Q4. Why can't you modify a REST response in `postHandle`?**
The body has usually already been written by the message converter, and the response may be committed. Use `ResponseBodyAdvice`.

**Q5. How do you register an interceptor for some paths only?**
`registry.addInterceptor(i).addPathPatterns("/api/**").excludePathPatterns("/api/auth/**")`.

**Q6. Why not store the start time in an interceptor field?**
Interceptors are singletons shared across threads. Store per-request data in request attributes.

**Q7. Filter or Interceptor for reading a custom annotation on the controller method?**
Interceptor: `((HandlerMethod) handler).getMethodAnnotation(MyAnno.class)`.

## Quick revision
- preHandle (forward, can stop) → controller → postHandle (reverse, success only) → afterCompletion (reverse, always).
- Register in `addInterceptors`; `@Component` alone isn't enough.
- `return false` must come with a written status/body. Design checks **fail-closed**.
- Per-request data goes in request attributes, not fields.
- Filter = servlet-level; Interceptor = MVC handler-level.
