# Database Authentication and Password Hashing

> **Source:** Lecture 36 (`Notes_Of_Aditya/Lecture 36/code/springPracticalDemo`)
> **Prerequisite:** [44_Spring Security Fundamentals](44_Spring%20Security%20Fundamentals.md) · **Next:** [46_Spring Security Architecture and Login Flow](46_Spring%20Security%20Architecture%20and%20Login%20Flow.md)

## Why this topic matters
"How do you store passwords?" is a guaranteed interview question. The answer is never "encrypted" and never "SHA-256". You must also design the user/role tables and explain why Spring Security needs `UserDetails`.

---

## 1. Authentication needs a source of truth
```
Claim (username) + credential (password)  ──verified against──►  trusted user data (DB / LDAP / IdP)
```
The default in-memory `user` isn't a user system. Real users (thousands, with roles and status) must survive restarts and scaling, so they go in a **database**.

## 2. Why `UserDetails`?
Every app's schema differs (`users`, `employees`, `members`; `email` vs `username`…). Spring Security defines **its own view** of a user and lets you adapt to it:
```java
public interface UserDetails {
    String getUsername();                         // login identifier (can be an email)
    String getPassword();                         // ENCODED password
    Collection<? extends GrantedAuthority> getAuthorities();
    boolean isAccountNonExpired();                // account validity (e.g. contract ended)
    boolean isAccountNonLocked();                 // temporary lock (e.g. too many failures)
    boolean isCredentialsNonExpired();            // password must be changed
    boolean isEnabled();                          // admin-disabled account
}
```
Your `User` entity (domain) ≠ `UserDetails` (security view). Connect them with an **adapter**; `UserDetailsService` is the bridge to your repository ([46](46_Spring%20Security%20Architecture%20and%20Login%20Flow.md)).
The login identifier **must be UNIQUE** (`@Column(unique = true)`), otherwise authentication is ambiguous.

## 3. Schema: users, roles, user_roles
```java
@Entity
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    @Column(nullable = false, unique = true) private String username;
    @Column(nullable = false) private String password;        // BCrypt hash, never raw
    private boolean enabled = true;

    @ManyToMany                                                // unidirectional: user → roles
    @JoinTable(name = "user_roles",
               joinColumns = @JoinColumn(name = "user_id"),
               inverseJoinColumns = @JoinColumn(name = "role_id"))
    private Set<Role> roles = new HashSet<>();
}

@Entity
public class Role {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    @Column(nullable = false, unique = true) private String name;   // "ROLE_USER", "ROLE_ADMIN"
}

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    boolean existsByUsername(String username);
}
public interface RoleRepository extends JpaRepository<Role, Long> {
    Optional<Role> findByName(String name);
}
```
> ⚠️ An entity named `User` maps to a table `user`, which is a **reserved word** in PostgreSQL/H2 and some MySQL contexts. Prefer `@Table(name = "users")`.

## 4. Password storage: why hashing (and which hash)
| Option | Verdict |
|---|---|
| Plain text | ❌ One DB leak exposes every password, and users reuse passwords elsewhere |
| Encryption (AES) | ❌ **Reversible**: DB + key = every password. You never need the original back |
| Fast hash (MD5/SHA-256) | ❌ Attackers test **billions of guesses/sec** offline; rainbow tables |
| **Adaptive, salted password hash**: **BCrypt**, SCrypt, Argon2, PBKDF2 | ✅ Deliberately slow (configurable work factor) + unique salt |

**Salt**: random data per password, so the same password gives different hashes, which defeats precomputed tables. BCrypt **stores the salt and cost inside the hash**: `$2a$10$<22-char salt><31-char hash>` (`2a` = version, `10` = cost 2¹⁰ rounds). No separate salt column.

### `PasswordEncoder`
```java
@Bean
public PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }   // or strength: new BCryptPasswordEncoder(12)

String hash = passwordEncoder.encode("secret123");          // $2a$10$A...  (different every call)
passwordEncoder.matches("secret123", hash);                 // true
passwordEncoder.matches("wrong", hash);                     // false
```
`matches()` **doesn't** re-encode and compare strings (a new salt would differ). It reads the salt and cost **from the stored hash** and re-computes.
Spring also offers `PasswordEncoderFactories.createDelegatingPasswordEncoder()`, which stores `{bcrypt}$2a$...` so you can upgrade algorithms later.

## 5. Registration flow
```
POST /register {username, password}
 → validate (@NotBlank, @Size(min=8)…)   ← policy: is the password acceptable?
 → reject duplicates (existsByUsername → 409)
 → passwordEncoder.encode(raw)           ← storage protection
 → assign default role ON THE SERVER (ROLE_USER), never from the request
 → save → return DTO (never the entity / hash)
```
```java
public User register(RegisterRequest request) {
    if (userRepository.existsByUsername(request.getUsername()))
        throw new IllegalArgumentException("Username is already registered");
    User user = new User();
    user.setUsername(request.getUsername());
    user.setPassword(passwordEncoder.encode(request.getPassword()));   // the critical line
    user.setEnabled(true);
    return userRepository.save(user);
}
```
The raw password lives only for this request. Never log it, return it, or store it.

## ⚠️ Issues in the lecture code (Lecture 36 project)
1. **`SecurityConfig` is entirely commented out** while `spring-boot-starter-security` is on the classpath. Boot's defaults apply: **every endpoint needs the generated `user` login**, including `/api/users/register`, and POSTs also need a CSRF token. So the register/login endpoints aren't reachable as intended. Lecture 37 adds the real config.
2. `AuthService` does `new BCryptPasswordEncoder()` itself instead of injecting the `PasswordEncoder` bean. That works, but it can't be configured or replaced, and it duplicates config.
3. **Hand-rolled `login()`** returns `Boolean` via `passwordEncoder.matches`. It bypasses Spring Security entirely: no `Authentication`, no session or token, no account-status checks, and `userOptional.get()` throws `NoSuchElementException` (500) for unknown users, which also reveals that a username doesn't exist. Use `AuthenticationManager` ([46](46_Spring%20Security%20Architecture%20and%20Login%20Flow.md), [47](47_JWT%20Authentication.md)).
4. No duplicate-username check or validation. A duplicate hits the UNIQUE constraint → `DataIntegrityViolationException` → 500.
5. Login reuses `UserRegisterRequestDto`. Use a separate `LoginRequest`.

---

## Interview Q&A
**Q1. How should passwords be stored?**
As an adaptive, salted, one-way hash (BCrypt/Argon2/SCrypt/PBKDF2) via `PasswordEncoder`. Never plain text or reversible encryption.

**Q2. Why not SHA-256?**
It's designed to be fast, so offline brute force is cheap. Password hashes must be slow and salted.

**Q3. What is a salt? Where does BCrypt keep it?**
Random per-password data that makes identical passwords hash differently. BCrypt embeds the salt and cost in the hash string.

**Q4. How does `matches()` work if each `encode()` result differs?**
It extracts the salt and cost from the stored hash, hashes the raw input with them, and compares.

**Q5. Why does Spring Security use `UserDetails` instead of your entity?**
To stay independent of your schema. You adapt your domain user to its contract.

**Q6. What's the difference between `isEnabled`, `isAccountNonLocked`, and `isCredentialsNonExpired`?**
Disabled by an admin / temporarily locked (e.g. brute force) / password must be renewed.

**Q7. Who decides a new user's role?**
The server (a default `ROLE_USER`). Never accept roles from the registration payload.

**Q8. What is `DelegatingPasswordEncoder`?**
Prefixes hashes with `{id}` so multiple algorithms can coexist and migrate over time.

## Quick revision
- Users + roles + user_roles; UNIQUE login identifier; `enabled` flag.
- Hash (BCrypt), don't encrypt; never a fast hash.
- `encode()` at registration, `matches()` at login (via Spring Security, not by hand).
- `UserDetails` = Spring's view of your user; adapt your entity to it.
- Validate input, block duplicates (409), and set roles server-side.
