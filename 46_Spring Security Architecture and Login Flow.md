# Spring Security Architecture and Login Flow

> **Source:** Lecture 37 (`Notes_Of_Aditya/Lecture 37/code/springPracticalDemo`)
> **Prerequisite:** [45_Database Authentication and Password Hashing](45_Database%20Authentication%20and%20Password%20Hashing.md) · **Next:** [47_JWT Authentication](47_JWT%20Authentication.md)

## Why this topic matters
"Explain the Spring Security authentication flow" means DelegatingFilterProxy → FilterChainProxy → filters → AuthenticationManager → ProviderManager → DaoAuthenticationProvider → UserDetailsService → PasswordEncoder → SecurityContext. It's a top senior-level question.

---

## 1. Security runs as servlet filters (before the controller)
```
Tomcat
 └─ DelegatingFilterProxy          (real servlet Filter; delegates to a Spring bean)
     └─ FilterChainProxy           (Spring Security's central filter, "springSecurityFilterChain")
         └─ picks the FIRST matching SecurityFilterChain
             ├─ SecurityContextHolderFilter
             ├─ CsrfFilter
             ├─ UsernamePasswordAuthenticationFilter   (form login POST /login)
             ├─ BasicAuthenticationFilter              (Authorization: Basic …)
             ├─ BearerTokenAuthenticationFilter        (JWT, see 47)
             ├─ AnonymousAuthenticationFilter
             ├─ ExceptionTranslationFilter             (AuthN error → 401/login; AccessDenied → 403)
             └─ AuthorizationFilter                    (requestMatchers rules)
 → DispatcherServlet → Controller
```
Order: **protection → authentication → exception handling → authorization**.
- **DelegatingFilterProxy** bridges the servlet container lifecycle and Spring beans.
- **FilterChainProxy** can hold several `SecurityFilterChain`s (e.g. `/api/**` with JWT, `/admin/**` with form login). **Only the first match runs**, so order them from most specific to most general, and keep a fallback `/**` chain.
- **HttpSecurity** is the builder; `http.build()` produces a `SecurityFilterChain`.
- An authentication filter only acts on requests it recognizes (e.g. `POST /login`). Other requests just pass through.

## 2. The authentication engine
```
Filter extracts credentials
  → UsernamePasswordAuthenticationToken.unauthenticated(username, rawPassword)   (a claim)
  → AuthenticationManager.authenticate(token)
      = ProviderManager → loops providers, picks one whose supports(token.class) is true
          → DaoAuthenticationProvider
               ├─ UserDetailsService.loadUserByUsername(username) → UserRepository → DB
               ├─ check enabled / locked / expired
               └─ PasswordEncoder.matches(raw, storedHash)
  → success: authenticated token (principal = UserDetails, credentials erased, authorities)
  → failure: AuthenticationException (UsernameNotFound → hidden as BadCredentials, Disabled, Locked…)
  → SecurityContextHolder.getContext().setAuthentication(result)
  → AuthorizationFilter → Controller
```
| Component | Role |
|---|---|
| `AuthenticationManager` | Entry point: `authenticate(Authentication)` |
| `ProviderManager` | Default impl; delegates to a list of `AuthenticationProvider`s |
| `AuthenticationProvider` | Mechanism-specific (`DaoAuthenticationProvider`, `JwtAuthenticationProvider`, LDAP, OTP…) |
| `UserDetailsService` | **Finder**: "given a username, load the user" |
| `UserDetails` | **Result**: one user's security view |
| `PasswordEncoder` | Verifies the password (`matches`) |
Form login and HTTP Basic use **different filters** but the **same** provider and DB check.

## 3. Lecture code
### Adapter: `CustomUserDetails`
```java
public class CustomUserDetails implements UserDetails {
    private final User user;
    public CustomUserDetails(User user) { this.user = user; }

    @Override public Collection<? extends GrantedAuthority> getAuthorities() {
        return user.getRoles().stream()
                .map(role -> new SimpleGrantedAuthority(role.getName()))   // "ROLE_USER"
                .toList();
    }
    @Override public String getPassword() { return user.getPassword(); }   // BCrypt hash
    @Override public String getUsername() { return user.getUsername(); }
    @Override public boolean isEnabled()  { return user.isEnabled(); }
    // isAccountNonExpired / NonLocked / CredentialsNonExpired → interface defaults (true)
}
```
(You can also build Spring's own `org.springframework.security.core.userdetails.User.withUsername(..).password(..).authorities(..).disabled(..).build()`.)

### Finder: `CustomUserDetailsService`
```java
@Service
public class CustomUserDetailsService implements UserDetailsService {
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User Not found"));
        return new CustomUserDetails(user);
    }
}
```
Roles are `@ManyToMany` (LAZY). The lecture's `UserRepository` uses `@EntityGraph(attributePaths = "roles")` on `findByUsername` so roles are loaded with the user (avoids `LazyInitializationException` outside a transaction).

### Wiring: `SecurityConfig`
```java
@Configuration
public class SecurityConfig {
    @Bean
    public PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }

    @Bean
    public DaoAuthenticationProvider authenticationProvider(CustomUserDetailsService uds, PasswordEncoder enc) {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider(uds);   // newer constructor form
        provider.setPasswordEncoder(enc);
        return provider;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http, DaoAuthenticationProvider provider) throws Exception {
        http.csrf(csrf -> csrf.disable())                    // demo only (Postman); keep it for browser/session apps
            .authenticationProvider(provider)
            .formLogin(Customizer.withDefaults())             // POST /login (session)
            .httpBasic(Customizer.withDefaults())             // Authorization: Basic
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/users/register").permitAll()
                .anyRequest().authenticated());
        return http.build();
    }
}
```
Registration assigns the default role on the server:
```java
Role role = roleRepository.findByName("ROLE_USER").get();     // ⚠️ see issues
user.getRoles().add(role);
```

### Authorization rules (examples)
```java
.requestMatchers("/admin/**").hasRole("ADMIN")            // checks authority ROLE_ADMIN
.requestMatchers(HttpMethod.DELETE, "/api/**").hasAuthority("USER_DELETE")
.requestMatchers("/error").permitAll()
.anyRequest().authenticated()
// method level: @EnableMethodSecurity + @PreAuthorize("hasRole('ADMIN')")
```

## 4. Registration vs login
| Registration | Login |
|---|---|
| Your controller/service | Spring Security filters + `AuthenticationManager` |
| `passwordEncoder.encode(raw)` | `passwordEncoder.matches(raw, hash)` |
| Store user + default role | Produce an authenticated `Authentication` |

## ⚠️ Issues in the lecture code (Lectures 37 and 38)
1. **Role bootstrap problem:** `register()` needs `ROLE_USER` to already exist (`findByName(..).get()` → `NoSuchElementException` → 500), but `/api/roles` (to create roles) requires authentication, and you can only authenticate after registering. Seed roles at startup (`CommandLineRunner`, `data.sql`, or a Flyway migration).
2. **Any authenticated user can `POST /api/roles`** and create roles. That should be `hasRole("ADMIN")`.
3. No duplicate-username check or validation on register (UNIQUE violation → 500). Returns 200 instead of 201.
4. CSRF disabled together with form login + sessions means browser sessions are exposed to CSRF. Disabling is only fine for pure bearer-token APIs (Lecture 38).
5. `/error` isn't permitted, so real error statuses can be masked as 401 ([44](44_Spring%20Security%20Fundamentals.md)).

---

## Interview Q&A
**Q1. Explain the Spring Security authentication flow.**
Filter extracts credentials → unauthenticated token → `AuthenticationManager` (ProviderManager) → matching provider (DaoAuthenticationProvider) → `UserDetailsService` loads the user, `PasswordEncoder.matches`, status checks → authenticated token → `SecurityContextHolder` → authorization → controller.

**Q2. DelegatingFilterProxy vs FilterChainProxy?**
DelegatingFilterProxy is a servlet filter registered with Tomcat that delegates to a Spring bean. FilterChainProxy is that bean; it selects and runs the matching `SecurityFilterChain`.

**Q3. Can there be multiple SecurityFilterChains?**
Yes, with `securityMatcher` + `@Order`. The first match handles the request; order from specific to general.

**Q4. `AuthenticationManager` vs `AuthenticationProvider`?**
The manager is the entry point and coordinator (ProviderManager). Providers implement specific mechanisms and declare `supports()`.

**Q5. `UserDetails` vs `UserDetailsService`?**
Result (one user's security view) vs finder (how to load a user by username).

**Q6. What does `ExceptionTranslationFilter` do?**
Converts `AuthenticationException` into a 401/login redirect (entry point) and `AccessDeniedException` into a 403 (or a login prompt for anonymous users).

**Q7. Why might "User not found" and "bad password" return the same error?**
`hideUserNotFoundExceptions` (default true) converts `UsernameNotFoundException` into `BadCredentialsException`, preventing username enumeration.

**Q8. How do you get the logged-in user in a controller?**
An `Authentication` parameter, `@AuthenticationPrincipal UserDetails`, or `SecurityContextHolder.getContext().getAuthentication()`.

## Quick revision
- Tomcat → DelegatingFilterProxy → FilterChainProxy → first matching SecurityFilterChain.
- Filter → token → AuthenticationManager/ProviderManager → DaoAuthenticationProvider → UserDetailsService + PasswordEncoder.
- Success → SecurityContextHolder → AuthorizationFilter → controller.
- UserDetails = adapter; UserDetailsService = finder.
- Seed roles; protect admin endpoints; keep CSRF for session apps.
