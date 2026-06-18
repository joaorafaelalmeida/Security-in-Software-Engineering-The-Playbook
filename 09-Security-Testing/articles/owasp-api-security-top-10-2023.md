# OWASP API Security Top 10 (2023)

> The reference list of the most critical API security risks — and the backbone for mapping manual web/API penetration tests.

## Metadata

- **Author(s):** OWASP API Security Project (Erez Yalon, Inon Shkedy, et al.)
- **Publication:** OWASP Foundation
- **Date:** 2023 (current edition; succeeds the 2019 edition)
- **Reading Time:** ~45 minutes (full list with descriptions)
- **Level:** Intermediate
- **Link:** <https://owasp.org/API-Security/editions/2023/en/0x11-t10/>

## Summary

APIs are now the dominant attack surface of modern web applications, and their risks differ enough from the
classic web-app risks that OWASP maintains a dedicated **API Security Top 10**. The 2023 edition refines the
2019 list, most notably splitting authorization into object-level, property-level, and function-level
categories, and adding risks around business-flow abuse and unsafe consumption of third-party APIs.

The list is not just a catalogue — it is a **testing checklist**. The reference PortSwigger lab series used
throughout this chapter maps cleanly onto it: every lab is an instance of one of these risks, which is why the
list anchors the **[penetration-testing methodology](../best-practices/web-application-penetration-testing-methodology.md)**.

The 2023 Top 10:

1. **API1 — Broken Object Level Authorization (BOLA):** the #1 risk; endpoints expose object ids without verifying ownership.
2. **API2 — Broken Authentication:** weak/incorrect verification of tokens and credentials.
3. **API3 — Broken Object Property Level Authorization (BOPLA):** merges excessive data exposure + mass assignment.
4. **API4 — Unrestricted Resource Consumption:** missing rate/size limits (DoS, cost).
5. **API5 — Broken Function Level Authorization:** access to privileged functions/verbs.
6. **API6 — Unrestricted Access to Sensitive Business Flows:** automatable abuse of legitimate flows (e.g. logic flaws).
7. **API7 — Server-Side Request Forgery (SSRF):** the API fetches an attacker-chosen URL.
8. **API8 — Security Misconfiguration.**
9. **API9 — Improper Inventory Management:** undocumented/shadow/old API versions.
10. **API10 — Unsafe Consumption of APIs:** trusting third-party API data implicitly.

## The Top 10 in Depth

Each entry below follows the same shape: *what it is → impact → how to test → how to defend*, with a link to
the grounding PortSwigger lab where the contributor's portfolio exercises it.

### API1:2023 — Broken Object Level Authorization (BOLA)

- **What it is.** An endpoint exposes an object identifier (`/orders/{id}`, `?user=…`) and authorizes by
  *session/role* but never checks that the caller owns the specific object. The 2019 list called the same idea
  IDOR; "BOLA" reframes it for object-rich API surfaces.
- **Impact.** Horizontal escalation (read/modify another tenant's data) — often *bulk* exploitable by
  enumerating ids, turning one bug into a full-database breach. Ranked #1 because it is ubiquitous, high-impact,
  and invisible to scanners (every request is a valid HTTP 200).
- **How to test.** With two accounts, replay every object-scoped request as the other user; change/enumerate
  ids (including UUIDs harvested from other responses). Access granted = finding.
- **How to defend.** Derive the subject from the session and enforce an ownership/relationship check on *every*
  object access, deny-by-default; treat unpredictable ids as defence-in-depth only.
- **Reference lab.** [User ID controlled by request parameter](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter).

### API2:2023 — Broken Authentication

- **What it is.** Flaws in *who you are*: weak/again-usable credentials, missing or incorrect token
  verification, guessable reset tokens, no protection against credential stuffing.
- **Impact.** Account takeover and impersonation — frequently of privileged users.
- **How to test.** Probe token verification (does the server trust the JWT `alg`? does it accept an empty/forged
  token?), reset flows (is the token bound to the account, single-use, high-entropy?), and brute-force/lockout
  behaviour.
- **How to defend.** Pin the JWT algorithm to an allow-list; make tokens the single source of truth for
  identity (never a sibling `email`/`username` field); rate-limit and use strong, rotated secrets.
- **Reference labs.** [JWT algorithm confusion](https://portswigger.net/web-security/jwt/algorithm-confusion/lab-jwt-authentication-bypass-via-algorithm-confusion),
  [OAuth implicit-flow bypass](https://portswigger.net/web-security/oauth/lab-oauth-authentication-bypass-via-oauth-implicit-flow),
  [password reset broken logic](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic).

### API3:2023 — Broken Object Property Level Authorization (BOPLA)

- **What it is.** The 2023 edition *merges* two 2019 entries: **Excessive Data Exposure** (responses serialise
  internal properties the client shouldn't see) and **Mass Assignment** (requests bind properties the client
  shouldn't write). Both are property-level authorization failures.
- **Impact.** Privilege escalation (`isAdmin=true`), financial abuse (`discount=100`), and PII leakage — and the
  two halves *chain*: a leaked field on the way out tells the attacker exactly what to write on the way in.
- **How to test.** Diff documented request fields against the response body for leaked properties; replay writes
  adding privileged/undocumented properties and check whether they take effect.
- **How to defend.** Bind request bodies to an explicit DTO/allow-list (never the domain object); serialise
  responses through an explicit output schema; keep the OpenAPI spec strict.
- **Reference lab.** [Exploiting a mass assignment vulnerability](https://portswigger.net/web-security/api-testing/lab-exploiting-mass-assignment-vulnerability).

### API4:2023 — Unrestricted Resource Consumption

- **What it is.** Missing limits on the resources a request can consume — CPU, memory, bandwidth, third-party
  spend (SMS/email), or records returned. (2019: "Lack of Resources & Rate Limiting.")
- **Impact.** Denial of service and *denial of wallet* — large, unpaginated queries, unbounded uploads, or
  enumeration that runs up cloud/third-party bills.
- **How to test.** Remove/inflate pagination params, send oversized payloads, and hammer expensive endpoints;
  watch for missing rate limits, timeouts, and size caps.
- **How to defend.** Enforce pagination, payload-size and timeout limits, per-client rate limiting/quotas, and
  spend caps on costly downstream actions.

### API5:2023 — Broken Function Level Authorization (BFLA)

- **What it is.** A user can invoke a *function/verb* outside their role — e.g. a regular user reaching an admin
  endpoint or using `DELETE`/`PUT` where only `GET` was intended. Distinct from BOLA: it's about the
  *operation/role*, not a specific object.
- **Impact.** Vertical privilege escalation to administrative capability.
- **How to test.** Map admin/privileged routes (often guessable: `/admin/...`, `/api/v1/internal/...`) and the
  HTTP methods each accepts, then call them as a low-privilege user.
- **How to defend.** Deny-by-default function authorization enforced centrally (gateway/middleware), with
  explicit role checks per route and method.

### API6:2023 — Unrestricted Access to Sensitive Business Flows

- **What it is.** *New in 2023.* A legitimate business flow (checkout, signup, ticket purchase, comment) lacks
  protection against *automated* abuse, or trusts client-supplied values it should own.
- **Impact.** Scalping/inventory hoarding, fraud, spam, and the classic logic flaws — e.g. trusting a
  client-sent `price` so an item is bought far below cost. No malicious characters; only a human judges the flow
  as *wrong*, which is why scanners miss it.
- **How to test.** Identify high-value flows and ask "what if this is automated, replayed, reordered, or has a
  value tampered?"; lower a client-supplied price/quantity, skip a step, or script the flow.
- **How to defend.** Recompute trusted values server-side; add anti-automation (device fingerprinting,
  CAPTCHAs, rate/velocity limits) and server-side consistency checks across multi-step flows.
- **Reference lab.** [Excessive trust in client-side controls](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-excessive-trust-in-client-side-controls).

### API7:2023 — Server-Side Request Forgery (SSRF)

- **What it is.** *Promoted to its own entry in 2023.* The API fetches a remote resource from a client-supplied
  URL without validating the destination, so the server makes attacker-directed requests.
- **Impact.** Reaching internal-only services, the cloud metadata endpoint (`169.254.169.254`) to steal IAM
  credentials, and pivoting through the server's trusted network position.
- **How to test.** Point any URL parameter at `localhost`/`127.0.0.1`/link-local/private ranges; test redirect
  following and DNS-rebinding; use an out-of-band collaborator for blind SSRF.
- **How to defend.** Allow-list destinations (scheme/host/port), re-check the *resolved IP*, disable redirect
  following, and add network egress filtering; never treat "came from localhost" as authentication.
- **Reference lab.** [Basic SSRF against the local server](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-localhost).

### API8:2023 — Security Misconfiguration

- **What it is.** Insecure defaults and gaps across the stack: verbose errors, missing security headers,
  permissive CORS, unpatched components, debug endpoints, or unhardened parsers (XXE lives here).
- **Impact.** Ranges from information disclosure to full compromise, depending on what's exposed.
- **How to test.** Inspect headers/CORS, error verbosity, default credentials, exposed admin/actuator/debug
  routes, and parser configuration (XML external entities).
- **How to defend.** Harden by default, automate configuration review, disable unused features, and keep
  components patched; disable DTD/external-entity resolution on XML parsers.

### API9:2023 — Improper Inventory Management

- **What it is.** Not knowing your full API surface: undocumented/shadow endpoints, deprecated `/v1` versions
  left running, non-production hosts exposed. (2019: "Improper Assets Management.")
- **Impact.** Old/unmaintained versions reintroduce already-fixed bugs and widen the attack surface invisibly.
- **How to test.** Enumerate versions/hosts (`/v1`,`/v2`,`staging.`,`dev.`), diff documented vs. live routes,
  and probe deprecated endpoints for weaker controls.
- **How to defend.** Maintain a current API inventory/OpenAPI catalogue, retire old versions, and gate
  non-production environments.

### API10:2023 — Unsafe Consumption of APIs

- **What it is.** *New in 2023.* Trusting data from third-party/upstream APIs more than user input — skipping
  validation, following their redirects blindly, or relying on transport security alone.
- **Impact.** Injection/SSRF/poisoning that enters through a "trusted" integration rather than the front door.
- **How to test.** Review every outbound integration: is upstream data validated and sanitised the same as user
  input? Are redirects followed? Is TLS verified?
- **How to defend.** Validate and sanitise third-party responses, restrict/disallow blind redirect following,
  enforce TLS verification, and isolate integrations.

## Key Takeaways

1. **Authorization is the dominant API risk.** Three of the ten (API1, API3, API5) are authorization failures,
   and BOLA tops the list. These are *semantic* bugs no scanner reliably finds — they demand manual testing.
2. **API3 has two halves that chain.** Excessive data exposure (the response leaks an internal field) makes
   mass assignment (writing that field) trivial — "the GET feeds the POST."
3. **Business-flow abuse (API6) is a first-class risk**, recognising that a perfectly valid sequence of
   requests can be malicious (logic flaws, automation abuse).
4. **SSRF earned its own entry (API7),** reflecting how often APIs fetch URLs on the client's behalf.
5. **The list doubles as a coverage map** for penetration tests and for CI security gates.

## Why This Matters

Most automated tooling is tuned for the *web* Top 10 (injection, XSS, misconfiguration) and underperforms on
the *API* Top 10's authorization and logic categories. Using the API Top 10 as an explicit checklist ensures
testers don't skip the highest-impact, least-automatable risks. It also gives developers and security reviewers
a shared vocabulary (BOLA vs BOPLA vs BFLA) precise enough to drive concrete fixes.

## Practical Applications

- **Scope coverage:** tick each category off during an engagement; the reference labs show what each looks like
  in practice (API1 → IDOR lab, API2 → JWT/OAuth/reset labs, API3 → mass-assignment lab, API6 → business-logic
  lab, API7 → SSRF lab).
- **Triage & severity:** classify findings against the list for consistent reporting.
- **Secure design review:** check new endpoints against API1/API3/API5 *before* they ship.
- **CI gating:** drive authorization regression tests (per the testing best practices) from the categories.

## Notable Quotes

> "Object level authorization is an access control mechanism that is usually implemented at the code level to
> validate that a user can only access the objects that they should have access to." — *API1:2023*

> "APIs tend to expose endpoints that handle object identifiers, creating a wide attack surface of object level
> access control issues." — *API1:2023*

## Related Topics

- [Web Application Penetration Testing Methodology](../best-practices/web-application-penetration-testing-methodology.md)
- [Access Control & API Authorization Testing](../best-practices/access-control-and-api-authorization-testing.md)
- [Authentication & Identity Testing](../best-practices/authentication-and-identity-testing.md)
- [Broken Access Control & API Authorization (defense)](../../02-Secure-Coding/best-practices/broken-access-control-and-api-authorization.md)

## Further Reading

- [OWASP API Security Top 10 2023 (full)](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [OWASP API Security Project](https://owasp.org/www-project-api-security/)
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [OWASP Top 10 2021 (web)](https://owasp.org/Top10/)

## Critical Analysis

The 2023 split of authorization into BOLA/BOPLA/BFLA is the edition's biggest improvement: it names exactly the
distinctions testers must make in practice, and matches how real bugs present (object id vs. privileged
property vs. privileged function). The main limitation is shared with any "top 10": it is a prioritisation aid,
not exhaustive coverage — and because its highest-ranked risks are precisely the ones automation can't catch,
the list is only as valuable as the *manual* testing discipline applied alongside it.

## Reference Labs

Each category below maps to a PortSwigger Web Security Academy lab solved as part of the contributor's
portfolio — concrete instances of the abstract risks:

- **API1 BOLA** → [User ID controlled by request parameter](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter)
- **API2 Broken Authentication** → [JWT algorithm confusion](https://portswigger.net/web-security/jwt/algorithm-confusion/lab-jwt-authentication-bypass-via-algorithm-confusion), [OAuth implicit flow](https://portswigger.net/web-security/oauth/lab-oauth-authentication-bypass-via-oauth-implicit-flow), [password reset broken logic](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic)
- **API3 BOPLA** → [Mass assignment](https://portswigger.net/web-security/api-testing/lab-exploiting-mass-assignment-vulnerability)
- **API6 Sensitive Business Flows** → [Excessive trust in client-side controls](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-excessive-trust-in-client-side-controls)
- **API7 SSRF** → [Basic SSRF against the local server](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-localhost)

## Tags

`owasp` `api-security` `api-top-10` `bola` `bopla` `authorization` `ssrf` `reference`

---

**Contributed by:** @roldao04
**Last Updated:** 2026-06-17
**Relevance:** High
