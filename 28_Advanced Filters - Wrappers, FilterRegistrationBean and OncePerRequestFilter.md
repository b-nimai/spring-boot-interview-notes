# Advanced Filters: Wrappers, FilterRegistrationBean and OncePerRequestFilter

> **Source:** Lecture 20 (`Notes_Of_Aditya/Lecture 20/code/filterDemo`)
> **Prerequisite:** [27_Servlet Filters - Basics](27_Servlet%20Filters%20-%20Basics%2C%20FilterChain%20and%20Order.md)

## Why this topic matters
Reading a request body in a filter, changing the response body, registering a filter only for `/api/*`, and `OncePerRequestFilter` are where real projects and senior interviews go after the basics.

---

## 1. What can a filter change?
| Thing | Read | Change |
|---|---|---|
| Response **headers / status** | — | ✅ easy (`setHeader`, `setStatus`), **before commit** |
| Response **body** | ❌ no `getBody()` | Needs `ContentCachingResponseWrapper` |
| Request **headers / params** | ✅ `getHeader`, `getParameter` | ❌ no setters, so use an `HttpServletRequestWrapper` |
| Request **body** | ⚠️ stream, readable **once** | Cache it + use a wrapper |

## 2. Response headers and status
```java
@Override
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
    HttpServletResponse httpRes = (HttpServletResponse) res;
    httpRes.setHeader("x-request-id", UUID.randomUUID().toString());   // replaces existing value
    // httpRes.addHeader("X-Feature", "Students");                     // adds another value
    chain.doFilter(req, res);
}
```
- **Committed response:** once downstream code flushes the body, the status and headers have already been sent, and later changes are ignored. Check with `httpRes.isCommitted()`. Set headers **before** the chain.

Blocking with a JSON error (lecture's `AuthenticationFilter`):
```java
if (token == null || !token.equals("12345")) {
    res.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
    res.setContentType("application/json");
    res.getWriter().write("{ \"message\" : \"Authentication is required\" }");
    return;                                   // no chain.doFilter → stops here
}
chain.doFilter(req, res);
```

## 3. Modifying the response body: `ContentCachingResponseWrapper`
`HttpServletResponse` is a **write-only destination** (`getWriter()` / `getOutputStream()` stream data toward the client). To read or replace the body afterwards, wrap it so the body is buffered:

```
Filter wraps response → passes WRAPPER down → controller writes into wrapper's buffer
      → filter reads buffer → replaces it → copyBodyToResponse() → client
```
```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
    ContentCachingResponseWrapper wrapped = new ContentCachingResponseWrapper((HttpServletResponse) res);

    chain.doFilter(req, wrapped);                                  // 1. downstream writes into the wrapper

    String original = new String(wrapped.getContentAsByteArray(), StandardCharsets.UTF_8);  // 2. read
    String modified = """
            { "originalResponse": %s, "appName": "Student Management System" }
            """.formatted(original);

    wrapped.resetBuffer();                                         // 3. clear body (keeps status + headers)
    wrapped.setContentType("application/json");
    wrapped.getOutputStream().write(modified.getBytes(StandardCharsets.UTF_8));
    wrapped.copyBodyToResponse();                                  // 4. ESSENTIAL: copy to real response
}
```
Response for `POST /api/students`:
```json
{ "originalResponse": {"message":"Student is saved successfully","name":"Aditya"}, "appName": "Student Management System" }
```
> ⚠️ **Forget `copyBodyToResponse()` and the client receives an empty body.**
> ⚠️ **Bug in the lecture's text block:** `"appName" : "Student Management System` is missing the closing quote, so the output is **invalid JSON**. The lecture also builds the string with the platform charset (`new String(bytes)`); always pass `UTF_8`.

**Limitations:** buffers the whole body in memory (bad for large or streaming/file responses), assumes the body is valid JSON, can invalidate `Content-Length`/`ETag`/compression. Build JSON with Jackson, not string concatenation.
**Better tool for wrapping controller responses:** `ResponseBodyAdvice` (Spring MVC).

## 4. Request body: a stream you can read only once
```java
// Lecture's RequestFilter
BufferedReader reader = httpRequest.getReader();
StringBuilder body = new StringBuilder();
String line;
while ((line = reader.readLine()) != null) body.append(line);
System.out.println(body);
chain.doFilter(req, res);
```
Why it breaks: an `InputStream` has a **position**. After the filter reads to the end, Jackson finds nothing left:
```
HTTP 400 — Required request body is missing
```
- There's no `getBody()`; the body might be JSON, XML, an image, or a 2 GB video, so it's exposed as **bytes in a stream**.
- `getReader()` and `getInputStream()` are two views of the **same** stream, and you can't mix them on one request.
- Containers don't auto-rewind, because buffering every body would cost memory and latency and create DoS risk.

**Fix:** read the bytes once, then pass a wrapper that returns a **fresh stream** each time:
```java
public class CachedBodyRequestWrapper extends HttpServletRequestWrapper {
    private final byte[] body;
    public CachedBodyRequestWrapper(HttpServletRequest req) throws IOException {
        super(req);
        this.body = req.getInputStream().readAllBytes();
    }
    @Override public ServletInputStream getInputStream() {
        ByteArrayInputStream in = new ByteArrayInputStream(body);     // new stream, position 0
        return new ServletInputStream() {
            public int read() { return in.read(); }
            public boolean isFinished() { return in.available() == 0; }
            public boolean isReady() { return true; }
            public void setReadListener(ReadListener l) { throw new UnsupportedOperationException(); }
        };
    }
    @Override public BufferedReader getReader() {
        return new BufferedReader(new InputStreamReader(getInputStream(), StandardCharsets.UTF_8));
    }
}
// in the filter:
CachedBodyRequestWrapper wrapped = new CachedBodyRequestWrapper(httpRequest);
log(new String(wrapped.getInputStream().readAllBytes(), UTF_8));
chain.doFilter(wrapped, res);                // controller can still read the body
```
Spring also ships `ContentCachingRequestWrapper`, but it caches **as the body is consumed downstream**, so you can only read the cache *after* `chain.doFilter`.
⚠️ Watch size limits and multipart uploads, and **never blindly log bodies** (passwords, tokens, personal data).

## 5. Changing request headers or params: `HttpServletRequestWrapper`
There's no `request.setHeader()` or `setParameter()`. Wrap the request and override the getters:
```java
class HeaderOverrideWrapper extends HttpServletRequestWrapper {
    HeaderOverrideWrapper(HttpServletRequest r) { super(r); }
    @Override public String getHeader(String name) {
        return "X-Tenant".equalsIgnoreCase(name) ? "default" : super.getHeader(name);
    }
    // for consistency also override getHeaders(name) and getHeaderNames()
}
chain.doFilter(new HeaderOverrideWrapper(httpRequest), res);
```
For parameters, override **all** of `getParameter`, `getParameterMap`, `getParameterValues`, `getParameterNames`. Spring data binding may use the map instead of `getParameter`.

## 6. Registering filters
### Option 1: `@Component`
Simple. Boot auto-registers it for **all URLs (`/*`)**. Order it with `@Order`.

### Option 2: `FilterRegistrationBean` (explicit control)
```java
@Configuration
public class FilterConfig {
    @Bean
    public FilterRegistrationBean<DummyFilter> dummyFilterRegistration() {
        FilterRegistrationBean<DummyFilter> reg = new FilterRegistrationBean<>();
        reg.setFilter(new DummyFilter());          // DummyFilter must NOT also be @Component
        reg.addUrlPatterns("/api/*", "/admin/*");  // ✅ varargs: one pattern per argument
        reg.setOrder(1);
        reg.setName("dummyFilter");
        return reg;
    }
}
```
It controls URL patterns, order, name, dispatcher types, async support, and `setEnabled(false)`.

> ⚠️ **Bug in the lecture code (✅ verified by running it):**
> ```java
> registrationBean.addUrlPatterns("/api/*, /admin/*");   // ONE string containing a comma
> ```
> That registers a single literal pattern `"/api/*, /admin/*"`, which **matches no URL**. Running the app and calling `POST /api/students` returned 200, but `"Dummy filter called"` was **never printed**. Pass the patterns as separate arguments.

> ⚠️ Registering the same filter with **both** `@Component` and `FilterRegistrationBean` creates two registrations, so it runs twice.

### Option 3: Check the URL inside the filter
```java
String uri = httpRequest.getRequestURI();
if (!uri.startsWith("/api/")) {
    chain.doFilter(req, res);
    return;                                   // ← REQUIRED
}
System.out.println("API-specific logic");
chain.doFilter(req, res);
```
> ⚠️ **The lecture's `DummyFilter` omits that `return`.** For non-`/api/` URLs it calls `chain.doFilter` **twice**, so the rest of the chain and the controller run twice, and the second pass usually fails because the response is already committed. (Based on Servlet-spec behaviour; it wasn't triggered in the lecture app because the broken URL pattern above means the filter never runs.)

## 7. `OncePerRequestFilter` (Spring)
```java
public class SpringDemoFilter extends OncePerRequestFilter {

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {   // true = SKIP this request
        return !request.getRequestURI().startsWith("/api/");
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,       // already HTTP types, no casting
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        filterChain.doFilter(request, response);
    }
}
```
> Note: in the lecture, `SpringDemoFilter` has no `@Component` or registration bean, so **it is never registered** and never runs.

**Why it exists:** one request can pass through the filter chain more than once via internal **dispatches** (`FORWARD`, `INCLUDE`, `ERROR` to `/error`, `ASYNC`). `OncePerRequestFilter` marks the request with an attribute so its logic runs **once per request** (with hooks such as `shouldNotFilterAsyncDispatch()` / `shouldNotFilterErrorDispatch()`).

| | `Filter` | `OncePerRequestFilter` |
|---|---|---|
| From | Jakarta Servlet | Spring |
| Types | `ServletRequest` (cast needed) | `HttpServletRequest` |
| Method | `doFilter` | `doFilterInternal` |
| Skip hook | — | `shouldNotFilter()` |
| Re-dispatch protection | Manual | Built-in |

Spring Security's filters and most custom JWT filters extend `OncePerRequestFilter`.

## 8. Choosing the right layer
| Need | Better tool |
|---|---|
| Authentication/authorization | Spring Security |
| Wrap all controller responses in an envelope | `ResponseBodyAdvice` |
| Error response format | `@RestControllerAdvice` |
| JSON field conversion | Jackson config / custom (de)serializer |
| Custom controller argument | `HandlerMethodArgumentResolver` |
| Input validation | Bean Validation |
| Business transformation | Service layer |
| Low-level HTTP concern (trace ID, logging, encoding, CORS) | **Filter** |

---

## Interview Q&A
**Q1. How do you read a response body in a filter?**
Wrap the response in `ContentCachingResponseWrapper`, pass the wrapper down the chain, read `getContentAsByteArray()`, then **always** call `copyBodyToResponse()`.

**Q2. Why can't you read the request body twice?**
It's a one-way `InputStream`. Once consumed, it's at EOF. Cache the bytes and supply an `HttpServletRequestWrapper` that returns a new `ByteArrayInputStream`.

**Q3. How do you add or modify a request header?**
You can't on the original request. Wrap it in `HttpServletRequestWrapper` and override `getHeader`/`getHeaders`/`getHeaderNames`.

**Q4. How do you apply a filter only to `/api/*`?**
`FilterRegistrationBean.addUrlPatterns("/api/*")`, or `shouldNotFilter()` in a `OncePerRequestFilter`.

**Q5. What is `OncePerRequestFilter` and why use it?**
A Spring base class guaranteeing one execution per request across internal dispatches (forward/error/async), with HTTP types and a `shouldNotFilter` hook.

**Q6. What does "response committed" mean?**
Status and headers have been sent to the client. Changing them afterwards has no effect, so set headers before `chain.doFilter`.

**Q7. What happens if a filter calls `chain.doFilter` twice?**
Downstream processing (including the controller) runs twice, and the second write typically fails with `IllegalStateException` because the response is already committed.

**Q8. Risks of buffering bodies?**
Memory use (large uploads), latency, broken streaming, stale `Content-Length`, and sensitive data in logs.

## Quick revision
- Headers/status: set before the chain. Body: use a wrapper.
- Response body: `ContentCachingResponseWrapper` → read → `resetBuffer` → write → **`copyBodyToResponse()`**.
- Request body is read-once; cache it + `HttpServletRequestWrapper` to re-read.
- `FilterRegistrationBean`: one URL pattern **per argument**; don't also add `@Component`.
- Manual URL check: `chain.doFilter(); return;`
- `OncePerRequestFilter` + `shouldNotFilter()`: Spring's preferred filter base.
