# Spring Security Fundamentals

> **Source:** Lecture 35 (`Notes_Of_Aditya/Lecture 35/code/springSecurityDemo`)
> **Next:** [45_Database Authentication and Password Hashing](45_Database%20Authentication%20and%20Password%20Hashing.md) · [46_Spring Security Architecture](46_Spring%20Security%20Architecture%20and%20Login%20Flow.md) · [47_JWT Authentication](47_JWT%20Authentication.md)

## Why this topic matters
Authentication vs authorization, the security context, sessions vs tokens, and "why do I get 401/403?" are everyday interview and debugging topics.

---

## 1. Security mindset
- **Asset**: anything valuable. Data (passwords, PII), operations (transfer money, delete account), infrastructure (DB, cloud), security material (JWT keys, session IDs), **availability**.
- **Threat**: something that could harm an asset. **Vulnerability**: a weakness. Threat + vulnerability = possible attack.
- **Trust boundary**: where data crosses trust levels (browser → server, server → DB, service → service). **The browser is untrusted**: hiding the "Delete" button isn't security; anyone can `curl DELETE /api/users/57`. **Enforce on the server.**
Spring Security covers authentication, authorization, the security context, session security, and attack protections (CSRF, headers). It does **not** fix SQL injection, insecure uploads, weak business rules, leaked secrets, or vulnerable dependencies.

## 2. Authentication vs authorization
| Authentication | Authorization |
|---|---|
| *Who are you? Prove it.* | *What may you do?* |
| Claim (username) + credential (password/OTP/key) → verified identity | Subject + action + resource + context → allow/deny |
| Fails → **401 Unauthorized** | Fails → **403 Forbidden** |
Factors: something you **know** (password), **have** (phone/OTP, key), **are** (biometrics). MFA = different categories.
**Authentication failure ≠ attack.** One wrong password is a failure; thousands is brute force. With credential stuffing, auth *succeeds* with stolen passwords. Respond generically ("Invalid username or password") and log details internally.

## 3. Identity model
| Term | Meaning |
|---|---|
| Principal | The current identity (username → `UserDetails`) |
| Credentials | Proof (password, token), erased after authentication |
| **GrantedAuthority** | A permission string: `COURSE_READ`, `ROLE_ADMIN`, `SCOPE_orders.read` |
| Role | Named group of permissions; stored as an authority prefixed `ROLE_` (`hasRole("ADMIN")` checks `ROLE_ADMIN`) |
| **Authentication** | Before: an *unauthenticated request* (principal = "aditya", credentials = "pwd", authenticated = false). After: a *verified principal* (UserDetails, authorities, authenticated = true) |
| **SecurityContext** | Holds the current `Authentication` |
| **SecurityContextHolder** | Static access, **ThreadLocal** by default; cleared after each request so pooled threads don't leak identities |
```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
auth.getName(); auth.getAuthorities();
// or in a controller: public String me(Authentication auth) / @AuthenticationPrincipal UserDetails user
```
Authorities must come from a **trusted source** (DB, verified token, IdP), never from client JSON like `{"role":"ADMIN"}`.

## 4. Stateful vs stateless authentication
| Stateful (session) | Stateless (token) |
|---|---|
| Login once → `SecurityContext` stored in `HttpSession` → `Set-Cookie: JSESSIONID=ABC123` | Each request carries `Authorization: Bearer <token>` |
| Cookie holds only an **ID**; state is on the server | Server validates the token and rebuilds `Authentication` per request |
| Easy server-side logout/invalidation | Revocation is harder (short expiry, deny-lists) |
| Needs shared session storage or sticky sessions when scaled | Scales horizontally easily |
- A session cookie is a **bearer credential**: steal it and you're the user. Use HTTPS, `Secure`, `HttpOnly`, `SameSite`, expiry, and **session-ID rotation on login** (session fixation protection, on by default).
- **Self-contained token** (JWT, signed claims, validated locally) vs **opaque token** (random string, looked up/introspected).
- **"Stateless" ≠ no data**: users, roles, keys, and refresh tokens still exist. It means *no HTTP login session is needed to recognize the next request*.
- **Mechanism ≠ session policy.** Form login/Basic/JWT say *how credentials arrive*; `SessionCreationPolicy` says *whether the context is persisted*. A JWT can be used statefully, and a cookie can carry a stateless token.

## 5. Spring Boot's defaults
Adding `spring-boot-starter-security`:
```java
// effective default SecurityFilterChain
http.authorizeHttpRequests(a -> a.anyRequest().authenticated())
    .formLogin(Customizer.withDefaults())
    .httpBasic(Customizer.withDefaults());
// + CSRF on, session-based, security headers
```
- **Default user** `user` with a random password logged at startup ("Using generated security password: …", development only), stored in an `InMemoryUserDetailsManager` with `ROLE_USER`.
- Override it with properties (lecture):
  ```properties
  spring.security.user.name=aditya
  spring.security.user.password=secret123
  spring.security.user.roles=USER,ADMIN
  ```
- **Back-off rules**: define a `SecurityFilterChain` bean and Boot's default *access rules* go away. Define a `UserDetailsService`/`AuthenticationProvider`/`AuthenticationManager` and the *generated user* goes away. These are independent.

## 6. ✅ Verified by running the lecture app
| Request | Result |
|---|---|
| `GET /api/students` (no auth, API client) | **401** |
| Same with `Accept: text/html` (browser) | **302 → /login** (form login page) |
| `GET` with Basic `aditya:secret123` | 200 "Student retrieved successfully" |
| Wrong password | 401 |
| `POST /api/students` with Basic, **no CSRF token** | **401** (see below) |
| `GET /api/students/csrf` | `{"headerName":"X-CSRF-TOKEN","parameterName":"_csrf","token":"…"}` |
| `POST` with session cookie + `X-CSRF-TOKEN` header | 200 "Student created successfully" |

### CSRF and the 401 surprise
CSRF protection is on by default. State-changing requests (POST/PUT/PATCH/DELETE) need a valid CSRF token tied to the session.
The PDF says a missing token gives **403**. In the debug log, `CsrfFilter` did respond 403 ("Invalid CSRF token found… Responding with 403"). But the container then **forwards to `/error`**, which is also secured, and that forwarded request is anonymous. So the client finally sees **401**.
Fix: `requestMatchers("/error").permitAll()`, after which the real 403 is returned. Lesson: **a 401/403 on a POST may be CSRF, not credentials or roles.**

The CSRF-token endpoint (lecture):
```java
@GetMapping("/csrf")
public CsrfToken getToken(CsrfToken csrfToken) { return csrfToken; }   // resolved from the request attribute
```
**When to disable CSRF:** stateless APIs authenticated by an `Authorization` header (browsers don't attach that automatically). Keep it for cookie/session-based browser apps.

---

## Interview Q&A
**Q1. Authentication vs authorization?**
Verifying identity vs deciding permissions. Failures give 401 vs 403 respectively.

**Q2. What's in an `Authentication` object?**
Principal, credentials, authorities, details, and the authenticated flag. It represents a request before verification and a verified principal after.

**Q3. What is `SecurityContextHolder` and how does it store data?**
The access point for the current `SecurityContext`, ThreadLocal by default, cleared at the end of each request.

**Q4. What does Spring Boot configure when you add the security starter?**
Every endpoint requires auth, form login + HTTP Basic, CSRF, a generated in-memory user `user` with a logged password, and security headers.

**Q5. Stateful vs stateless authentication?**
Session (server stores context, cookie holds the ID) vs token (client sends a credential each request; the server validates it and rebuilds auth).

**Q6. What is CSRF, and when can you disable protection?**
Cross-Site Request Forgery: a malicious site makes the victim's browser send an authenticated request (cookies attach automatically). Protection is unnecessary for pure bearer-header APIs.

**Q7. `hasRole("ADMIN")` vs `hasAuthority("ADMIN")`?**
`hasRole` adds the `ROLE_` prefix (checks `ROLE_ADMIN`). `hasAuthority` matches the exact string.

**Q8. Why can a valid user get 403?**
Missing role/authority, **missing CSRF token**, or method security denial.

## Quick revision
- Enforce on the server; the browser is untrusted.
- AuthN = who (401) vs AuthZ = what (403).
- Authentication → SecurityContext → SecurityContextHolder (ThreadLocal).
- Session (JSESSIONID) vs bearer token; mechanism ≠ session policy.
- Boot defaults: everything authenticated, form + Basic, CSRF on, generated `user`.
- CSRF failure → 403, but shows as 401 unless `/error` is permitted (verified).
