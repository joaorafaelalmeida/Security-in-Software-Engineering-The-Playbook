# Zero Trust for a CRUD Web API: From Perimeter Trust to Per-Request Verification

> Stop trusting a request because of where it came from. Verify identity and authorization on *every* request to *every* resource, and you collapse a whole family of access-control bugs into one architectural decision.

## The Problem

Most web applications are still built around an implicit-trust perimeter. A request that
gets past the outer wall (the reverse proxy, the firewall, the "it's only on the internal
network" assumption) is treated as trustworthy for every hop afterwards. The controllers
trust the gateway, the services trust the controllers, the repository trusts the service,
and the database trusts whatever connects to it. There is exactly one trust boundary, drawn
around the whole application, and crossing it once buys access to everything inside.

This is easy to spot in a threat model. When the Level-1 Data Flow Diagram has a *single*
trust boundary wrapping the controllers, services, and data layer together, every internal
data flow is implicitly trusted. Input validation at the edge is the only gate, and it only
answers "is this well-formed?" — never "who are you?" or "are you allowed to do this?".

What goes wrong when trust is granted by network position rather than verified per request:

- **Anonymous read of the entire dataset.** A list endpoint such as
  `GET /resources?limit=500` returns everything to anyone who can reach it. (OWASP
  A01:2021 — Broken Access Control.)
- **Unauthorized writes.** `POST`, `PUT`, and `DELETE` execute with no proof of identity and
  no role check. An attacker enumerates IDs via the open read endpoint, then deletes each
  record; cascading foreign keys quietly take the children with them.
- **No attribution.** With no authenticated principal and no audit log, destructive actions
  are unattributable — you cannot answer "who did this?" because there is no "who".
- **Cross-origin theft.** A permissive CORS policy (`allowedOriginPatterns("*")` *with*
  `allowCredentials(true)`) lets a malicious page read credentialed responses from a victim's
  browser. The request is trusted purely because of where it claims to come from — the exact
  location-based trust Zero Trust rejects. (OWASP A05:2021 — Security Misconfiguration.)
- **Lateral movement.** Once inside the network segment, an attacker reaches the database
  directly (e.g. plaintext `:3306` with a leaked root password) because "same network = trusted".

These look like five or six independent vulnerabilities. They are not. They are five or six
symptoms of **one** root cause: *trust is granted by position, not verified by evidence*.
Patch them one at a time and you will keep finding more. Fix the architecture once and the
whole class closes.

> A useful tell during review: if a team writes abuse stories demanding "a token on every
> endpoint", "an authorized role on every write", "least-privilege scoping on every read",
> and "an immutable audit log", they have independently re-derived the NIST SP 800-207 Zero
> Trust tenets without naming them. This entry names the pattern so the fix becomes one
> decision instead of six tickets.

## The Solution

### Overview

**Zero Trust** (NIST SP 800-207) replaces the perimeter with per-request verification:
*never trust, always verify*. Its three principles are **verify explicitly**, **least
privilege**, and **assume breach**. The architectural move is to put a **Policy Enforcement
Point (PEP)** in front of every resource and have it consult a **Policy Decision Point (PDP)**
before *any* access is allowed. Tenet 6 of SP 800-207 states it plainly: *all resource
authentication and authorization are dynamic and strictly enforced before access is allowed.*

Zero Trust is a **property of the architecture, not a product**. OIDC, JWT, OPA, mTLS and
SPIFFE are tools that help you implement it; none of them *is* Zero Trust, and adding any one
of them does not make a system "zero trust". Equally, Zero Trust is *not* "no trust" and *not*
"MFA everywhere" — it is dynamic, evidence-based trust, re-evaluated continuously.

For a typical CRUD API the migration is a small, ordered set of steps. The first three are
the high-value core and are achievable in a single iteration; the rest are defense-in-depth
and maturity targets.

### Implementation

#### Step 1: Put every endpoint behind a Policy Enforcement Point (default-deny)

Add a security filter that intercepts every request and denies by default. The *last* matching
rule must be "everything else requires authentication", so a newly added endpoint is protected
the moment it ships, not the moment someone remembers to protect it. This filter chain *is*
your PEP — the gate the single all-encompassing trust boundary was missing.

```java
// Spring Security: default-deny PEP, stateless (token-based)
@Bean
SecurityFilterChain api(HttpSecurity http) throws Exception {
    return http
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/actuator/health").permitAll()
            .anyRequest().authenticated())   // default-deny floor — keep this LAST
        .build();
}
```

#### Step 2: Authenticate every request against a central identity provider

Stop owning passwords. Externalize authentication to an IdP (Keycloak, Auth0, Entra ID, ...)
and have the API validate a signed token on each request — checking issuer, audience, and
expiry. Policy lives outside the application code; the app only verifies evidence.

```java
// Spring Security as an OAuth2 Resource Server (validates iss, aud, exp, nbf)
http.oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
```

```properties
spring.security.oauth2.resourceserver.jwt.issuer-uri=https://idp.example.com/realms/app
spring.security.oauth2.resourceserver.jwt.audiences=my-api
```

#### Step 3: Authorize per-resource, not per-session (least privilege)

A token proves *who* you are; it does not say *what* you may do. Authorize each request against
the *specific resource* it targets. Coarse role checks (BFLA defense) stop someone from calling
an admin-only function; object-level scoping (BOLA defense) stops an authenticated user from
reaching another user's records by guessing an ID.

```java
@Configuration
@EnableMethodSecurity                       // must sit on a @Configuration class to take effect
class MethodSecurityConfig { }

@PreAuthorize("hasRole('ADMIN')")          // function-level authz (BFLA)
public void deleteResource(Long id) { ... }

// object-level authz (BOLA): scope the query to the caller, never trust the path id alone
public Resource read(Long id, Jwt caller) {
    return repo.findByIdAndOwner(id, caller.getSubject())
               .orElseThrow(NotFoundException::new);
}
```

#### Step 4: Restrict origins, encrypt traffic, and ship secure headers

Replace wildcard CORS with an explicit allow-list, never `"*"` together with credentials.
Enable TLS, connect to the database with a least-privilege account over an encrypted channel
(not `root` / `useSSL=false`), and emit a Content-Security-Policy. Most security headers come
"for free" once the filter chain is in place.

```java
@Bean
CorsConfigurationSource cors() {
    var cfg = new CorsConfiguration();
    cfg.setAllowedOrigins(List.of("https://app.example.com"));  // explicit origin
    cfg.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    cfg.setAllowCredentials(true);                              // safe: origin is not "*"
    var src = new UrlBasedCorsConfigurationSource();
    src.registerCorsConfiguration("/**", cfg);
    return src;
}
```

#### Step 5: Emit telemetry and re-evaluate continuously

Log every authorization decision and every state change to an immutable, append-only trail,
attributed to the authenticated principal. Keep token lifetimes short (5-15 min) so trust is
re-checked frequently rather than granted once. This turns "who did this?" from unanswerable
into a query.

```java
@EnableJpaAuditing(auditorAwareRef = "auditor")
class AuditConfig {
    @Bean
    AuditorAware<String> auditor() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
                             .map(Authentication::getName);
    }
}
```

#### Step 6 (maturity target): Authenticate the workload, not just the user

For higher maturity, give services and the database their own cryptographic identities
(X.509 / SPIFFE) and require mutual TLS east-west, so "on the same network" stops meaning
"trusted". This is the heaviest lift and is honestly an *Advanced/Optimal* goal, not a
day-one requirement.

## Code Examples

### Bad Practice (Vulnerable)

```java
// Implicit-trust perimeter: anonymous CRUD + wildcard CORS with credentials
@RestController
@RequestMapping("/resources")
class ResourceController {

    @GetMapping
    List<Resource> list(@RequestParam(defaultValue = "100") int limit) {
        return service.findAll(limit);            // anyone reads the full dataset
    }

    @DeleteMapping("/{id}")
    void delete(@PathVariable Long id) {
        service.delete(id);                       // anyone deletes any record, untraced
    }
}

@Configuration
class CorsConfig implements WebMvcConfigurer {
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOriginPatterns("*")       // wildcard origin...
                .allowedMethods("*")
                .allowCredentials(true);          // ...WITH credentials -> credentialed theft
    }
}
```

**Why this is problematic:**

- No `SecurityFilterChain`, no authentication, no `@PreAuthorize` — every endpoint is fully
  anonymous (OWASP A01).
- `allowedOriginPatterns("*")` deliberately sidesteps Spring's guard against `"*"` +
  credentials, so any site can read credentialed responses (OWASP A05).
- No principal and no audit log means destructive actions cannot be attributed to anyone.
- Trust is granted by network reachability; cross every gate once and you own everything.

### Good Practice (Secure)

```java
// Per-request verification: default-deny PEP + token auth + per-resource authz + audit
@Bean
SecurityFilterChain api(HttpSecurity http) throws Exception {
    return http
        .cors(Customizer.withDefaults())                                  // explicit allow-list bean
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))      // validate iss/aud/exp
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/actuator/health").permitAll()
            .anyRequest().authenticated())                                // default-deny floor
        .build();
}

@RestController
@RequestMapping("/resources")
class ResourceController {

    @GetMapping
    List<Resource> list(@AuthenticationPrincipal Jwt caller,
                        @RequestParam(defaultValue = "100") int limit) {
        return service.findOwnedBy(caller.getSubject(), limit);           // least-privilege scoping
    }

    @PreAuthorize("hasRole('ADMIN')")                                     // per-request authz
    @DeleteMapping("/{id}")
    void delete(@PathVariable Long id) {
        service.delete(id);                                               // audited via JPA auditing
    }
}
```

**Why this works:**

- No request reaches a handler unauthenticated — the PEP verifies explicitly, every time.
- Authorization is decided per request against the targeted resource (function-level *and*
  object-level), enforcing least privilege.
- The audit listener attributes every state change to a real principal, satisfying
  non-repudiation.
- CORS trusts a named origin, not a location wildcard. Network position grants nothing.

> **Note for reviewers:** a default JWT-to-authorities converter emits `SCOPE_*` authorities,
> so `hasRole('ADMIN')` will silently never match a `roles` claim. If your IdP issues roles,
> map them explicitly (`JwtGrantedAuthoritiesConverter` with the right claim name and the
> `ROLE_` prefix). This is the single most common reason a "secured" endpoint stays wide open.

## Benefits

- **Collapses a vulnerability class, not a vulnerability.** One architectural change
  (verify-per-request) closes anonymous reads, unauthorized writes, IDOR/BOLA, and BFLA at once.
- **Secure by default.** A default-deny floor means new endpoints are protected the moment they
  are added, eliminating "forgot to secure it" regressions.
- **Attribution and forensics.** Every state change is tied to an identity, so incidents become
  answerable instead of mysterious.
- **Limits blast radius.** A single compromised credential or service cannot pivot to everything,
  because trust is re-checked at each resource.
- **Decouples policy from code.** Externalized identity and (optionally) externalized policy let
  security evolve without redeploying the application.

## Common Pitfalls

1. **Treating a tool as the goal.** "We added Spring Security / a JWT library, so we're zero
   trust." No — Zero Trust is an architectural property. A misconfigured filter chain that
   permits all is no better than none.
2. **Authenticating but not authorizing per object.** A valid token is not permission. Without
   object-level scoping you swap anonymous access for *any-authenticated-user* access — still
   broken (BOLA).
3. **Default `permitAll` matcher first / catch-all last that allows.** Order matters. The final
   matcher must deny by default (`anyRequest().authenticated()`). One stray `permitAll("/**")`
   reopens everything.
4. **Wildcard CORS with credentials.** `allowedOriginPatterns("*")` plus `allowCredentials(true)`
   is a deliberate bypass of the framework's own guard. Always name explicit origins.
5. **Claim-mapping mismatch.** Roles in the token under a custom claim won't satisfy
   `hasRole(...)` unless you map them. Test with a real token, not a mock.
6. **Mistaking network controls for per-request authz.** Segmentation and mTLS are
   defense-in-depth; SP 800-207A is explicit that identity-based enforcement should be
   *augmented* by network location, never *replaced* by it. Network controls complement, they
   do not substitute.
7. **Calling rate limiting "Zero Trust".** Rate limiting and other availability/DoS controls are
   valuable defense-in-depth, but availability is *outside* the SP 800-207 tenet set (which
   covers authentication, authorization, posture, and telemetry). Present it as complementary
   hardening, not as a ZT pillar.

## When to Apply

- **Always:** Any API or service that exposes data or state-changing operations over a network —
  which is essentially every web/REST/GraphQL API. The default-deny PEP and per-request
  authentication (Steps 1-3) should be considered table stakes.
- **Recommended:** Multi-tenant systems, anything handling personal or regulated data (health,
  finance), and microservice architectures where lateral movement is a real risk.
- **Consider:** Workload identity and mTLS (Step 6) when you are maturing beyond the basics, or
  in zero-trust network segments. Acknowledge these as Advanced/Optimal maturity targets;
  CISA's Zero Trust Maturity Model expects gradual, per-pillar advancement, not a big bang.

## Standards & Compliance

- **NIST SP 800-207 (Zero Trust Architecture):** Tenets 1-7; especially Tenet 2 (all
  communication secured regardless of network location), Tenet 3 (per-session, least
  privilege), Tenet 6 (auth/authz strictly enforced before access), and Tenet 7 (telemetry).
- **NIST SP 800-207A:** Identity-based segmentation for cloud-native apps; recommends
  end-user-to-resource authorization (object-level authz) and augmenting identity with network
  signals rather than relying on either alone.
- **OWASP Top 10 2021 — A01: Broken Access Control:** the class this practice closes
  (missing access controls, IDOR/BOLA, function-level bypass/BFLA).
- **OWASP Top 10 2021 — A05: Security Misconfiguration:** permissive CORS, missing security
  headers.
- **OWASP API Security Top 10 2023 — API1 (BOLA), API5 (BFLA):** the precise API-level
  vocabulary for the authorization gaps Step 3 addresses.
- **CISA Zero Trust Maturity Model v2.0:** pillars (Identity, Networks, Applications & Workloads,
  Data) and the Traditional -> Initial -> Advanced -> Optimal progression used to plan adoption.
- **CWE-862 (Missing Authorization), CWE-639 (Authorization Bypass Through User-Controlled Key),
  CWE-942 (Permissive Cross-domain Security Policy with Untrusted Domains).**

## Verification & Testing

### Manual Checks

- Hit every endpoint with **no token** and confirm `401`.
- Hit a write/admin endpoint with a **valid but under-privileged token** and confirm `403`.
- Request **another principal's object** by ID with a valid token and confirm `404/403`, not the
  object (BOLA check).
- Inspect the CORS response: no `Access-Control-Allow-Origin: *` paired with
  `Access-Control-Allow-Credentials: true`.

### Automated Testing

```java
@Test
void anonymous_request_is_denied() throws Exception {
    mockMvc.perform(get("/resources")).andExpect(status().isUnauthorized());
}

@Test
void user_cannot_delete() throws Exception {
    mockMvc.perform(delete("/resources/1").with(jwt().authorities("ROLE_USER")))
           .andExpect(status().isForbidden());
}
```

### Security Scanning

- **DAST (e.g. OWASP ZAP):** must be **authentication-configured** to be meaningful — an
  unauthenticated scan against an unauthenticated API reports "no auth issues" because it never
  tries to authenticate. Verify the scanner imports the *correct* OpenAPI path for your
  framework (e.g. springdoc serves `/v3/api-docs`, not `/openapi.json`), or the API surface is
  silently missed.
- **SAST (e.g. Semgrep):** rules for missing `@PreAuthorize`, `permitAll`/wildcard matchers, and
  permissive CORS configurations.

> Access control gaps are largely *invisible to scanners by default*. They are caught by
> design-time threat modeling, then confirmed by **authenticated** dynamic tests. Do not
> conclude "the scan is clean, so we're fine".

## Related Best Practices

- **Chapter 01 — Planning and Threat Modeling:** the single-trust-boundary DFD and STRIDE-derived
  abuse stories are how you *discover* implicit trust before writing code.
- **Chapter 09 — Security Testing:** authenticated DAST and negative authorization test cases are
  how you *prove* the PEP works.
- **Chapter 11 — Compliance and Standards:** maps this practice to NIST 800-207, OWASP, and CISA.
- **Chapter 05 — Secrets Management:** the leaked-DB-credential lateral-movement path is partly a
  secrets problem; least-privilege DB accounts complement per-request authz.

## Further Reading

- NIST SP 800-207, *Zero Trust Architecture* (2020) — https://doi.org/10.6028/NIST.SP.800-207
- NIST SP 800-207A, *A Zero Trust Architecture Model for Access Control in Cloud-Native
  Applications in Multi-Location Environments* (2023) — https://doi.org/10.6028/NIST.SP.800-207A
- CISA, *Zero Trust Maturity Model v2.0* (2023) —
  https://www.cisa.gov/zero-trust-maturity-model
- OWASP Top 10 2021 — A01 Broken Access Control —
  https://owasp.org/Top10/A01_2021-Broken_Access_Control/
- OWASP Top 10 2021 — A05 Security Misconfiguration —
  https://owasp.org/Top10/A05_2021-Security_Misconfiguration/
- OWASP API Security Top 10 2023 (BOLA, BFLA) —
  https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- Spring Security Reference — OAuth2 Resource Server (JWT) —
  https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html
- Spring Security Reference — Method Security —
  https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html
- Spring Security Reference — CORS —
  https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html

## Tags

`zero-trust` `access-control` `authentication` `authorization` `bola` `bfla` `cors` `least-privilege` `nist-800-207` `owasp-a01` `owasp-a05` `api-security`

---

**Contributed by:** @sebastiaoteixeira
**Last Updated:** 2026-06-17
**Difficulty Level:** Intermediate
**Impact:** High
