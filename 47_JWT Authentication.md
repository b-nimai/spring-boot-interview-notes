# JWT Authentication (Stateless) with Spring Security

> **Source:** Lecture 38 (`Notes_Of_Aditya/Lecture 38/code/springPracticalDemo`)
> **Prerequisite:** [46_Spring Security Architecture and Login Flow](46_Spring%20Security%20Architecture%20and%20Login%20Flow.md) · **Next:** [48_OAuth2 and OpenID Connect](48_OAuth2%20and%20OpenID%20Connect%20-%20Google%20Login.md)

## Why this topic matters
JWT login is the default for REST APIs and mobile/SPA backends. Interviewers ask about JWT structure, signing vs encoding, HS256 vs RS256, statelessness, expiry and revocation, and how Spring validates tokens.

---

## 1. Why tokens? HTTP forgets
Authenticating one request doesn't authenticate the next.
- **Session**: the server stores the `SecurityContext` in `HttpSession` and the client sends `JSESSIONID` (stateful).
  `SessionCreationPolicy`: `ALWAYS` · `IF_REQUIRED` (default) · `NEVER` (don't create, use if present) · **`STATELESS`** (never use a session for security).
- **JWT**: the client sends a signed token holding claims, and the server rebuilds `Authentication` from it **per request** (stateless).

## 2. JWT structure
```
xxxxx.yyyyy.zzzzz  =  Base64URL(header) . Base64URL(payload) . Base64URL(signature)
```
```json
// header
{ "alg": "HS256", "typ": "JWT" }
// payload (claims)
{ "iss": "coderarmy-api", "sub": "aditya", "iat": 1786570000, "exp": 1786570900,
  "authorities": ["ROLE_USER"] }
```
| Claim | Meaning |
|---|---|
| `iss` | Issuer |
| `sub` | Subject (who) |
| `aud` | Intended audience (API) |
| `iat` / `exp` / `nbf` | Issued at / expires / not before |
| `jti` | Unique token id (useful for revocation) |
| custom | e.g. `authorities` |

### Encoding ≠ encryption ≠ signing
- **Base64URL is encoding.** Anyone can decode the header and payload. **Never put passwords, secrets, or sensitive PII in a JWT.**
- A plain hash isn't enough: an attacker edits the payload and recomputes `SHA256(payload)`.
- The **signature** uses a secret/private key: `HMAC-SHA256(secret, base64(header) + "." + base64(payload))`. Editing the payload (`ROLE_USER` → `ROLE_ADMIN`) without the key leaves `A.B2.C`, a mismatched signature, and the token is **rejected**. The signature gives **integrity + authenticity**; it doesn't hide data (JWE does encryption).

### HS256 vs RS256/ES256
| | HS256 (HMAC) | RS256 / ES256 |
|---|---|---|
| Keys | One **shared secret** signs and verifies | **Private** key signs, **public** key verifies |
| Verifier can mint tokens? | Yes (it has the secret) | No |
| Fit | Single app issues and validates (this lecture) | Auth server + many APIs (publish keys via JWKS) |
Pitfalls: weak or short secrets (HS256 needs ≥ 256-bit random keys), accepting `alg: none`, not validating `exp`/`iss`/`aud`.

## 3. Flow
```
LOGIN   POST /auth/login {username,password}
        → AuthenticationManager → DaoAuthenticationProvider (DB + BCrypt)   (same as 46)
        → JwtService → JwtEncoder → signed JWT → {"accessToken":"eyJ..."}

REQUEST GET /api/users/hello   Authorization: Bearer eyJ...
        → BearerTokenAuthenticationFilter → JwtAuthenticationProvider
        → JwtDecoder (verify signature, exp, iss) → JwtAuthenticationConverter (claims → authorities)
        → JwtAuthenticationToken in SecurityContext → authorization → controller   (NO DB lookup)
```
Two providers, two jobs: `DaoAuthenticationProvider` (username/password at login) and `JwtAuthenticationProvider` (bearer tokens afterwards). Both produce an `Authentication`.

## 4. Implementation (lecture code)
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>   <!-- Nimbus JWT + bearer filter -->
</dependency>
```
```properties
jwt.secret=${JWT_SECRET}      # Base64 of ≥32 random bytes, e.g. `openssl rand -base64 32` (lecture hard-codes it ⚠️)
jwt.issuer=coderarmy-api
jwt.expiry=900                # seconds = 15 min
```
### Keys, encoder, decoder
```java
@Bean
public SecretKey jwtSecretKey(@Value("${jwt.secret}") String secret) {
    return new SecretKeySpec(Base64.getDecoder().decode(secret), "HmacSHA256");
}

@Bean
public JwtEncoder jwtEncoder(SecretKey key) {                       // CREATE / SIGN
    return NimbusJwtEncoder.withSecretKey(key).algorithm(MacAlgorithm.HS256).build();
}

@Bean
public JwtDecoder jwtDecoder(SecretKey key, @Value("${jwt.issuer}") String issuer) {   // VERIFY / VALIDATE
    NimbusJwtDecoder decoder = NimbusJwtDecoder.withSecretKey(key).macAlgorithm(MacAlgorithm.HS256).build();
    decoder.setJwtValidator(JwtValidators.createDefaultWithIssuer(issuer));   // exp/nbf + iss
    return decoder;
}

@Bean
public AuthenticationManager authenticationManager(DaoAuthenticationProvider provider) {
    return new ProviderManager(provider);         // exposed so the login controller can call it
}
```
### Token creation: `JwtService`
```java
public String generateToken(Authentication authentication) {
    Instant now = Instant.now();
    List<String> authorities = authentication.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority).toList();             // ["ROLE_USER"]
    JwtClaimsSet claims = JwtClaimsSet.builder()
            .issuer(issuer).issuedAt(now).expiresAt(now.plusSeconds(expiry))
            .subject(authentication.getName())
            .claim("authorities", authorities)
            .build();
    return jwtEncoder.encode(JwtEncoderParameters.from(claims)).getTokenValue();
}
```
> On older Spring Security versions, `NimbusJwtEncoder` with a MAC key may need the header set explicitly: `JwtEncoderParameters.from(JwsHeader.with(MacAlgorithm.HS256).build(), claims)`. The lecture configures the algorithm on the encoder builder instead.

### Login endpoint
```java
@PostMapping("/auth/login")
public LoginResponseDto login(@RequestBody LoginRequestDto req) {
    Authentication request = UsernamePasswordAuthenticationToken.unauthenticated(req.getUsername(), req.getPassword());
    Authentication authentication = authenticationManager.authenticate(request);   // throws on bad creds
    return new LoginResponseDto(jwtService.generateToken(authentication));
}
```
### Mapping claims to authorities
By default the resource server reads `scope`/`scp` and prefixes `SCOPE_`. We store `"authorities": ["ROLE_USER"]`:
```java
@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtGrantedAuthoritiesConverter c = new JwtGrantedAuthoritiesConverter();
    c.setAuthoritiesClaimName("authorities");
    c.setAuthorityPrefix("");                         // keep "ROLE_USER" as-is → hasRole("USER") works
    JwtAuthenticationConverter conv = new JwtAuthenticationConverter();
    conv.setJwtGrantedAuthoritiesConverter(c);
    return conv;
}
```
### Filter chain
```java
http.csrf(csrf -> csrf.disable())                                    // OK: bearer header, not cookies
    .authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/users/register", "/auth/login").permitAll()
        .anyRequest().authenticated())
    .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
    .oauth2ResourceServer(o -> o.jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter)));
```
```java
@GetMapping("/api/users/hello")
public String sayHello(Authentication authentication) {
    return "Hello, you are logged in as : " + authentication.getName();   // = "sub"
}
```

## 5. Session vs JWT
| Session | JWT |
|---|---|
| Server stores state; client sends `JSESSIONID` | Token carries claims; server validates each request |
| Instant logout/revocation | Valid until `exp` unless you add a deny-list or short expiry + refresh tokens |
| Role changes apply immediately | Roles inside the token stay stale until it expires |
| CSRF relevant (cookie) | Not CSRF-prone with an Authorization header; **XSS** risk if stored in `localStorage` |

**Production practices:** short-lived access tokens (5–15 min) + **refresh tokens** (rotated, stored server-side or as `HttpOnly` cookies); validate `iss`/`aud`/`exp`; load the secret from env/vault; RS256 + JWKS when multiple services validate; HTTPS only; never log tokens.

## ⚠️ Issues in the lecture code
1. **Signing secret committed** in `application.properties` (`jwt.secret=n/BcSh5t7...`). Anyone with the repo can forge admin tokens. Use `${JWT_SECRET}`.
2. **No logout or revocation** and no refresh token. A stolen token works for its full 15 minutes.
3. Bad credentials at `/auth/login`: `AuthenticationException` escapes the controller. Map it explicitly to a 401 JSON body (`@ExceptionHandler(AuthenticationException.class)`), and permit `/error`.
4. Field injection (`@Autowired`) in `AuthController`/`JwtService`; prefer constructors. The unused `ResourceLoader` should go.
5. The issues from [46](46_Spring%20Security%20Architecture%20and%20Login%20Flow.md) remain: unseeded roles, `/api/roles` open to every user, no duplicate check on register.

---

## Interview Q&A
**Q1. What are the three parts of a JWT?**
Header (alg, typ), payload (claims), signature, each Base64URL-encoded and joined with dots.

**Q2. Is a JWT encrypted? Can a user read it?**
A signed JWT (JWS) isn't encrypted; anyone can read it. Only JWE encrypts. Don't store secrets in it.

**Q3. How does the server detect tampering?**
It recomputes or verifies the signature over header.payload with its key. Any change makes it fail.

**Q4. HS256 vs RS256?**
Shared symmetric secret vs private/public key pair. RS256 lets many services verify without being able to mint tokens.

**Q5. Why `SessionCreationPolicy.STATELESS`? Why disable CSRF?**
Each request authenticates via its bearer token, so no session is needed. CSRF exploits automatically-sent cookies, and an `Authorization` header isn't sent automatically.

**Q6. How do you log out or revoke a JWT?**
Short expiry + refresh-token revocation, a deny-list by `jti`, a token version per user, or rotating keys.

**Q7. JwtEncoder vs JwtDecoder vs JwtAuthenticationConverter?**
Create/sign; verify/validate (signature, exp, iss); map claims to `GrantedAuthority`s and the `Authentication`.

**Q8. Where should the client store the token?**
Memory or an `HttpOnly`+`Secure`+`SameSite` cookie (then handle CSRF). `localStorage` is exposed to XSS.

**Q9. Does a JWT request hit the database?**
Not for authentication: claims are trusted after signature validation. That's also why role changes lag until the token expires.

## Quick revision
- JWT = header.payload.signature (Base64URL) → readable, tamper-evident.
- Login: AuthenticationManager → JwtEncoder (sub, iss, iat, exp, authorities).
- Requests: Bearer → JwtDecoder (sig + exp + iss) → converter (prefix "") → SecurityContext.
- STATELESS + CSRF off (header tokens). HS256 = shared secret; RS256 = key pair.
- Short expiry, refresh tokens, secret from env, never sensitive data in claims.
