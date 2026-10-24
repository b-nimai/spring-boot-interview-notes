# OAuth 2.0 and OpenID Connect: "Login with Google"

> **Source:** Lecture 39 (`Notes_Of_Aditya/Lecture 39/code/oauthDemo`)
> **Prerequisite:** [47_JWT Authentication](47_JWT%20Authentication.md)

## Why this topic matters
"OAuth vs OIDC", "explain the authorization code flow", "what is PKCE", and "access token vs ID token" are classic security interview questions, and "Sign in with Google" is in almost every product.

---

## 1. The problem: delegated access
Your app needs a user's Google Calendar. **Never** ask for their Google password: they'd be sharing it, and you'd get access to everything (Gmail, Drive…).
**OAuth 2.0** = a framework for **delegated authorization**: the user signs in at Google and grants your app a **limited** access token.

## 2. Roles
| Role | Example | Does |
|---|---|---|
| Resource Owner | The user | Owns the data, grants consent |
| **Client** | Your Spring Boot app | Requests access on the user's behalf ("client" ≠ browser) |
| Authorization Server | Google accounts | Authenticates the user, shows consent, issues tokens |
| Resource Server | Google Calendar API | Hosts the data, validates access tokens |

**Scope** = what access is requested (`openid profile email`, `calendar.readonly`). **Consent** = the user approves those scopes. Ask for the **least privilege**.

## 3. OAuth vs OpenID Connect; three tokens
| | OAuth 2.0 | OpenID Connect (OIDC) |
|---|---|---|
| Purpose | Delegated **authorization** ("what can this app access?") | **Authentication / identity** on top of OAuth ("who signed in?") |
| Trigger | — | `openid` scope |

| Token | Sent to | Purpose |
|---|---|---|
| **Access token** | Resource server (`Authorization: Bearer`) | Call protected APIs. May be JWT or opaque; clients treat it as opaque |
| **ID token** (OIDC, signed JWT) | The **client** itself | Who authenticated: `sub`, `iss`, `aud`, `exp`, `name`, `email` |
| **Refresh token** | Authorization server's token endpoint | Get new access tokens without re-login |
Don't call APIs with the ID token, and don't use the access token as proof of identity.

## 4. Authorization Code Flow (+ PKCE)
```
1. Browser → GET /oauth2/authorization/google  (our app)
2. App → 302 to Google authorize endpoint:
      client_id, redirect_uri, response_type=code, scope=openid profile email,
      state=<random>, code_challenge=<SHA256(verifier)>   (PKCE)
3. User authenticates AT GOOGLE (password/passkey/2FA never touches our app)
4. User consents to scopes
5. Google → 302 to redirect_uri: /login/oauth2/code/google?code=A7D9...&state=<same>
6. App verifies state, then BACK-CHANNEL POST to Google token endpoint:
      code + redirect_uri + client_secret (+ code_verifier)
   ← access_token, id_token, (refresh_token)
7. App validates the ID token (signature via Google JWKS, iss, aud = our client_id, exp, nonce)
8. App creates its session → user logged in
```
- **Front channel** (via the browser): the authorization request, the code, the state. **Back channel** (server ↔ server): the code exchange and the tokens. Tokens never pass through browser redirects.
- **`state`** ties the callback to the original request and prevents login CSRF.
- **Authorization code** = short-lived and single-use. It's not a token; it gets exchanged.
- **PKCE** ("pixy"): the client creates a random `code_verifier`, sends `SHA256(verifier)` as the `code_challenge`, then proves possession at the token exchange. A stolen code is useless without the verifier. The **client secret** authenticates a confidential client; PKCE binds the code to the client *instance* that started the flow. Required for SPAs/mobile, recommended for everyone.
- `client_id` is public; `client_secret` is a secret.

## 5. Google Cloud setup
1. Create a project → Google Auth Platform → branding (app name, support email).
2. Audience: **External** (add test users while in testing mode).
3. Scopes: `openid`, `profile`, `email` only.
4. Clients → Create → **Web application**.
5. Authorized redirect URI **exactly** `http://localhost:8080/login/oauth2/code/google` (otherwise `redirect_uri_mismatch`). Spring's template: `{baseUrl}/login/oauth2/code/{registrationId}`.
6. Copy the client ID and secret into **environment variables**.

## 6. Spring Boot implementation (lecture code)
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security-oauth2-client</artifactId>   <!-- Boot 4 name; Boot 3: spring-boot-starter-oauth2-client -->
</dependency>
```
```properties
spring.security.oauth2.client.registration.google.client-id=${CLIENT_ID}
spring.security.oauth2.client.registration.google.client-secret=${CLIENT_SECRET}
spring.security.oauth2.client.registration.google.scope=openid,profile,email
# Google is a known provider → authorize/token/userinfo/jwks URIs are pre-configured
```
### Security config
```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http, CustomOidcUserService customOidcUserService) throws Exception {
    http.authorizeHttpRequests(auth -> auth.requestMatchers("/").permitAll().anyRequest().authenticated())
        .oauth2Login(oauth -> oauth
            .userInfoEndpoint(ui -> ui.oidcUserService(customOidcUserService))   // hook our signup logic
            .defaultSuccessUrl("/profile", true));
    return http.build();
}
```
`oauth2Login()` provides `/oauth2/authorization/{id}` (start), the `/login/oauth2/code/{id}` callback, the code exchange, ID token validation, and the UserInfo call.

### Hook: link the Google identity to an internal user
```java
@Service
public class CustomOidcUserService implements OAuth2UserService<OidcUserRequest, OidcUser> {
    private final OidcUserService delegate = new OidcUserService();      // let Spring do the OIDC work
    @Autowired private UserService userService;

    @Override
    public OidcUser loadUser(OidcUserRequest req) throws OAuth2AuthenticationException {
        OidcUser oidcUser = delegate.loadUser(req);                       // tokens exchanged + validated by now
        String provider = req.getClientRegistration().getRegistrationId();  // "google"
        userService.registerOrUpdate(provider, oidcUser);                 // our signup/update
        return oidcUser;
    }
}

@Entity
public class User {                      // no password column: Google owns credentials
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    private String name, email, provider, providerSubject;   // provider + sub = stable external identity
}
```
**Why `provider + sub`, not email?** Email can change, be missing, or be unverified. `sub` is the provider's stable user id. Make the pair **UNIQUE**.

### Profile endpoint
```java
@GetMapping("/profile")
public Map<String, Object> profile(@AuthenticationPrincipal OidcUser oidcUser) {
    User user = userService.findByProviderAndSubject("google", oidcUser.getSubject()).orElseThrow();
    return Map.of("internalUserId", user.getId(), "provider", user.getProvider(),
                  "subject", oidcUser.getSubject(), "name", oidcUser.getClaimAsString("name"),
                  "email", oidcUser.getClaimAsString("email"));
}
```
Identity claims come from Google; `internalUserId`, roles, and business data come from **our** DB.

**After login the app is session-based**: the `SecurityContext` lives in the HTTP session (`JSESSIONID`). The browser doesn't send the ID token on later requests.
| `oauth2Login()` | `oauth2ResourceServer().jwt()` |
|---|---|
| Logs a **browser user** into *this* app (session) | Protects an **API** that receives bearer access tokens |

## ⚠️ Issues in the lecture code
1. **`UserService.registerOrUpdate` isn't `@Transactional`.** For an existing user it calls setters on an entity loaded by a finished repository transaction (detached), so **name/email updates are never saved**. The PDF's version has `@Transactional`. Add it, or call `save()`.
2. `/profile` hard-codes `"google"` instead of using the authenticated provider (`OAuth2AuthenticationToken#getAuthorizedClientRegistrationId()`).
3. No UNIQUE constraint on `(provider, providerSubject)`, so concurrent first logins can create duplicates.
4. Field injection (`@Autowired`) and an entity named `User` (reserved word in some DBs; use `@Table(name = "app_users")`).
5. Secrets are correctly externalized (`${CLIENT_ID}`, `${CLIENT_SECRET}`). Never commit the real values.

---

## Interview Q&A
**Q1. OAuth 2.0 vs OpenID Connect?**
OAuth = delegated authorization (access tokens for APIs). OIDC = an identity layer on OAuth (ID token, `openid` scope, UserInfo) used for login.

**Q2. Explain the authorization code flow.**
Redirect to the auth server with client_id/redirect_uri/scope/state(/PKCE) → user authenticates and consents → code returned via the browser → backend exchanges the code (+ secret/verifier) for tokens over the back channel.

**Q3. Why not return tokens directly in the redirect (implicit flow)?**
Tokens in URLs leak (history, logs, referrer). The code exchange keeps tokens on the back channel. Implicit is deprecated.

**Q4. What is PKCE and why is it needed?**
Proof Key for Code Exchange: hash(verifier) is sent first and the verifier later, so an intercepted code can't be redeemed by an attacker.

**Q5. What is the `state` parameter for?**
It correlates request and callback and protects against CSRF/login-injection.

**Q6. Access token vs ID token vs refresh token?**
Call APIs / prove who logged in (for the client) / obtain new access tokens.

**Q7. How do you map a Google user to your own user table?**
Store provider + `sub` (unique). Create on first login, update profile claims later, and keep app roles in your DB.

**Q8. After Google login, how are later requests authenticated in Spring?**
Via the HTTP session (`JSESSIONID`) holding the `OAuth2AuthenticationToken`.

## Quick revision
- OAuth = delegated authorization; OIDC = login/identity (ID token).
- Roles: resource owner, client, authorization server, resource server; scopes + consent.
- Auth code flow: code via the front channel, tokens via the back channel; `state` + PKCE.
- Spring: `oauth2Login()` + custom `OidcUserService` → link provider + `sub` to an internal user.
- Session-based after login; resource-server JWT is a different use case.
