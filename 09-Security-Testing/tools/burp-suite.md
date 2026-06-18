# Burp Suite

> The de-facto intercepting proxy and manual web-application testing toolkit — the workbench for manual penetration testing.

## Overview

Burp Suite (by PortSwigger) is an integrated platform for testing web application security. It sits as a
**man-in-the-middle proxy** between the browser and the target, letting a tester observe, intercept, and modify
every HTTP/S request and response. Where automated **[DAST](../../06-Dynamic-Analysis-DAST)** scanners (e.g.
OWASP ZAP) excel at breadth and CI automation, Burp is the tool of choice for the **manual-first** workflow
that finds authorization, business-logic, and identity bugs — the ones that require a human to decide a valid
request is *wrong*. It is the primary tool behind the
**[Web Application Penetration Testing Methodology](../best-practices/web-application-penetration-testing-methodology.md)**.

Editions: **Community** (free; Proxy, Repeater, Decoder, limited Intruder), **Professional** (adds the active
scanner, full-speed Intruder, BApp store extensions), and **Enterprise** (CI/CD-oriented automated scanning).

## Key Features

- **Proxy:** intercept, view, and edit live traffic; build a site map of the target.
- **Repeater:** manually re-issue and iterate on a single request — the core loop for IDOR/logic/auth testing.
- **Intruder:** automate parameter fuzzing/enumeration (object-id enumeration, brute force, payload sweeps).
- **JWT Editor (extension):** decode, edit, and re-sign JWTs; derive a symmetric key from an RSA public key (algorithm-confusion attacks).
- **Collaborator:** detect out-of-band/blind interactions (blind SSRF, blind XXE, OOB injection).
- **Decoder / Comparer / Sequencer:** encoding transforms, response diffing, and session-token randomness analysis.
- **Match-and-replace & session-handling rules:** auto-rewrite headers/cookies (e.g. swap session cookies while browsing for authorization testing).

## Use Cases

- Manual exploitation of access-control (IDOR/BOLA) and business-logic flaws by replaying and tampering requests.
- Token attacks — JWT algorithm confusion, OAuth flow tampering — via the JWT Editor and Repeater.
- Server-side request manipulation (SSRF, XXE) and out-of-band detection with Collaborator.
- Enumerating object identifiers or hidden fields with Intruder.
- General reconnaissance: mapping the request surface and reading responses for leaked fields/gadgets.

## Installation & Setup

### Prerequisites
- Java runtime is bundled with the official installers (no separate JRE needed).
- A browser configured to trust Burp's CA certificate (Burp ships an embedded Chromium that is pre-configured).

### Installation

```bash
# Download the installer for your platform from the official site, then run it.
# https://portswigger.net/burp/releases
# On Linux:
sh ./burpsuite_community_linux_v*.sh

# Or launch the embedded browser from Burp: Proxy → Intercept → Open Browser
```

## Usage Examples

### Basic Usage

```text
1. Proxy → Intercept is on. Browse the target through Burp's browser.
2. Find the request of interest in Proxy → HTTP history.
3. Right-click → "Send to Repeater" (Ctrl+R).
4. In Repeater, modify a parameter (e.g. id=wiener → id=carlos) and click "Send".
5. Inspect the response for impact (another user's data, a leaked field, an error).
```

### Advanced Usage

```text
# Object-id enumeration with Intruder
1. Send the request to Intruder (Ctrl+I).
2. Mark the id value as a payload position: §id§.
3. Payloads → numbers/usernames list → Start attack.
4. Sort by response length to spot authorized vs. unauthorized objects.

# JWT algorithm confusion with the JWT Editor extension
1. Capture a valid JWT; fetch the public key from /jwks.json.
2. JWT Editor Keys → New RSA Key (paste JWK) → then derive a Symmetric Key
   whose k = base64url(PEM public key).
3. In Repeater's "JSON Web Token" tab, set alg=HS256 and sub=administrator,
   then "Sign" with the symmetric key.
```

### Configuration Example

```text
# Proxy → Match and replace (automate authorization testing):
#   Type: Request header
#   Match: ^Cookie:.*$
#   Replace: Cookie: session=<User B's session>
# Now browse as User A while requests are silently re-issued as User B.
```

## Pros and Cons

### Advantages ✅
- Best-in-class manual workflow (Proxy + Repeater) for human-driven testing.
- Rich extension ecosystem (BApp store): JWT Editor, Param Miner, Autorize, etc.
- Collaborator enables blind/out-of-band detection few other tools match.
- Industry standard — findings and workflows are widely understood.

### Limitations ⚠️
- The most useful automation (active scanner, full-speed Intruder) is **Professional-only** (paid license).
- Manual-first means it does not, by itself, give automated wide-coverage CI scanning (pair with ZAP/Enterprise).
- Steeper learning curve than a one-click scanner.

## Integration

How Burp fits into a security pipeline alongside automated tooling:

```yaml
# Manual Burp testing complements automated DAST in CI (e.g. OWASP ZAP / Burp Enterprise).
# Typical split:
#   - CI (every PR):      automated DAST baseline scan (broad, fast, low-FP gate)
#   - Pre-release (human): manual Burp penetration test (authorization, logic, identity)
#   - PoC scripts:         re-run in CI as authorization/logic regression tests
```

## Alternatives

- **OWASP ZAP** — free, automation/CI-friendly DAST; see **[Chapter 06](../../06-Dynamic-Analysis-DAST)**. Strong for breadth; weaker manual UX.
- **mitmproxy** — scriptable intercepting proxy; great for automation, no security-testing UI.
- **Caido** — newer Rust-based proxy with a modern UI; smaller extension ecosystem.

## Resources

- [Official Documentation](https://portswigger.net/burp/documentation)
- [Burp Suite releases / download](https://portswigger.net/burp/releases)
- [Web Security Academy (free labs)](https://portswigger.net/web-security)
- [BApp Store (extensions)](https://portswigger.net/bappstore)

## License

Commercial / Freemium — Community Edition is free; Professional and Enterprise are paid.

## Reference Labs

Burp features exercised across the contributor's PortSwigger Web Security Academy portfolio:

- **Repeater** — request tampering in the [IDOR](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter), [business-logic](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-excessive-trust-in-client-side-controls), and [mass-assignment](https://portswigger.net/web-security/api-testing/lab-exploiting-mass-assignment-vulnerability) labs.
- **JWT Editor** — RSA→HMAC key derivation and re-signing in the [JWT algorithm-confusion](https://portswigger.net/web-security/jwt/algorithm-confusion/lab-jwt-authentication-bypass-via-algorithm-confusion) lab.
- **Exploit server** — hosting and delivering the auto-submitting page in the [CSRF method-bypass](https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-validation-depends-on-request-method) lab.
- **Repeater (XML body)** — external-entity injection in the [XXE file-retrieval](https://portswigger.net/web-security/xxe/lab-exploiting-xxe-to-retrieve-files) lab; URL substitution in the [SSRF](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-localhost) lab.

## Tags

`burp-suite` `proxy` `manual-testing` `penetration-testing` `dast` `jwt` `intruder` `repeater`

---

**Contributed by:** @roldao04
**Last Updated:** 2026-06-17
