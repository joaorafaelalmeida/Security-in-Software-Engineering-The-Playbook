# Authentication & Identity Testing

> Trust-boundary attacks against tokens and identity flows — JWT algorithm confusion, OAuth implicit-flow bypass, and broken password-reset logic.

## The Problem

Modern authentication leans on tokens and federated identity (JWT, OAuth/OIDC, signed reset links). The bugs
are rarely in the cryptography — they are in the **last inch**, where a backend reads an identity claim from a
field it shouldn't trust, or fails to verify a token it should. These are **OWASP A07:2021 (Identification &
Authentication Failures)** and **API2:2023 (Broken Authentication)**.

A single rule explains every case in the reference series:

> **The component that issues an identity assertion is the only one allowed to interpret it — everyone
> downstream must re-verify, never re-trust.**

Three labs break that rule in three different ways. (Note: token *defenses* — e.g. JWT hardening — live in the
secure-coding chapter; this entry is the offensive/testing counterpart and deliberately does not duplicate
them.)

## The Solution

### Overview

Wherever **identity** and **proof** travel as *separate* values, attack the seam: keep a valid proof, change
the identity. Wherever the **token itself names how it should be checked**, attack the verifier's trust in
that self-description.

| Lab | Seam attacked | One-line technique |
|---|---|---|
| JWT algorithm confusion | Token chooses its own verification algorithm | Flip `alg` RS256→HS256; sign with the public key as the HMAC secret |
| OAuth implicit-flow bypass | Identity (`email`) sits beside the proof (`token`) | Keep your valid token, swap in the victim's email |
| Password reset broken logic | Identity (`username`) sits beside an unverified token | Empty the token, set `username=victim` |

### Implementation

#### Test 1: JWT algorithm confusion (RS256 → HS256)
The JWT header names the `alg`. A careless verifier trusts it. With RS256 the server signs with its RSA
*private* key and verifies with the *public* key; with HS256 it signs and verifies with one *symmetric secret*.
If the verifier lets the token pick HS256, it feeds the only key it has — the **public** RSA key — into HMAC as
the secret. Since the public key is published (e.g. at `/jwks.json`), the attacker knows the exact verification
bytes and can sign any payload.

1. Capture a valid session JWT; fetch the public key from `/jwks.json`.
2. In **[Burp's JWT Editor](../tools/burp-suite.md)**, import the RSA public key, then derive a *symmetric*
   key whose `k` is the base64url of the PEM public key.
3. Set the payload `sub` to `administrator`, header `alg` to `HS256`, and **Sign** with that symmetric key.

```text
header  {"kid":"…","alg":"HS256"}
payload {"iss":"portswigger","exp":…,"sub":"administrator"}
        → forged token accepted at /admin
```

The novelty is the inversion: publishing an RSA public key is supposed to be perfectly safe — *until an `alg`
you forgot to pin turns it into the HMAC password to your own admin panel.*

#### Test 2: OAuth implicit-flow authentication bypass
In the implicit flow (`response_type=token`) the access token is returned in the redirect fragment; the client
then tells its own backend who you are. If `/authenticate` trusts the `email` in the request body and never
re-binds it to the presented token, identity and proof are decoupled.

1. Complete the flow legitimately as yourself and capture the `POST /authenticate` the callback sends.
2. In **Burp Repeater**, change *only* the `email` to the victim's, leaving your valid `token` untouched:

```http
POST /authenticate HTTP/1.1
Content-Type: application/json

{"email":"victim@example.net","token":"<your-own-valid-access-token>"}
```

3. The server issues a session for the victim — account takeover with no password.

#### Test 3: Password-reset broken logic
The reset form submits a `temp-forgot-password-token` *and* a `username`. If the server decides whose password
to change from the `username` and never validates the token, the token is decorative.

1. Request a reset for your own account and capture the submission.
2. Resend with the **token emptied** and `username` set to the victim:

```http
POST /forgot-password?temp-forgot-password-token= HTTP/1.1
Content-Type: application/x-www-form-urlencoded

temp-forgot-password-token=&username=victim&new-password-1=pwned&new-password-2=pwned
```

3. Log in as the victim with the new password. Emptying the token and still succeeding *proves* it was never
   checked.

#### Test 4: Generalize — find the seam
For any auth/identity flow, list the fields and ask: which one is the **proof**, which is the **identity
claim**, and are they cryptographically bound? If you can hold the proof and change the claim, you have a
finding.

## Code Examples

### Bad Practice (the verifier logic you are testing for)

```python
# Token chooses the algorithm — algorithm confusion.
jwt.decode(token, key, algorithms=jwt.get_unverified_header(token)["alg"])

# Identity read from the request body next to an unverified credential.
def authenticate(body):
    create_session(email=body["email"])   # trusts client email; token never bound

def reset_password(req):
    user = find_user(req.form["username"]) # identity from username, token ignored
    user.set_password(req.form["new-password-1"])
```

**Why this is problematic:**
- `algorithms=<header alg>` lets the attacker select HS256 and reuse the public key as the secret.
- `create_session(email=body["email"])` trusts "the browser said this is carlos."
- `find_user(username)` makes the reset token decorative — any account is resettable.

### Good Practice (what a passing re-test confirms)

```python
jwt.decode(token, public_key, algorithms=["RS256"])   # pin the algorithm

def authenticate(body):
    claims = verify_token_serverside(body["token"])    # derive identity FROM the token
    create_session(email=claims["email"])

def reset_password(req):
    user = user_for_valid_token(req.form["token"])     # token is the single source of truth
    user.set_password(req.form["new-password-1"])
```

**Why this works:**
- An explicit `algorithms` allow-list makes the token unable to downgrade the verifier.
- Identity is derived from the verified token/claims, never from a sibling field.
- The reset token alone determines the account; an empty/invalid token fails closed.

## Benefits

- **Covers the auth bugs scanners miss** — the flaw is in trust placement, not in a detectable payload.
- **One transferable rule** (re-verify, never re-trust) generalizes across JWT, OAuth, SAML, reset links, and webhooks.
- **High impact** — every finding here is account takeover or privilege escalation.

## Common Pitfalls

1. **Assuming "it uses RSA/strong crypto" means it's safe** — algorithm confusion makes key strength irrelevant.
2. **Not testing the empty/null token** — the fastest proof that a token is unvalidated.
3. **Testing only the documented flow** — the bug is usually in the backend's *last* request (`/authenticate`), not the OAuth dance.
4. **Forgetting `kid`/`jku`/`x5u` header injection** as sibling JWT attacks worth probing.
5. **Treating these as crypto problems** — they are trust-boundary problems.

## When to Apply

- **Always:** any app using JWTs, OAuth/OIDC, social login, or self-service password reset.
- **Recommended:** whenever auth code, token verification, or identity-provider integration changes.
- **Consider:** continuous testing of token issuance/verification as part of identity-platform changes.

## Framework/Language-Specific Guidance

### Python
```bash
# requests + a JWT/JWK helper (e.g. cryptography for JWK→PEM) reproduces the alg-confusion forge.
pip install requests cryptography
```

### JavaScript/Node.js
```text
# Verify which library/option selects the algorithm (jsonwebtoken: always pass `algorithms`).
# Inspect the callback's server call — that is where implicit-flow trust usually breaks.
```

### Tooling
```text
# Burp Suite JWT Editor: import JWK, derive symmetric key from public key, re-sign.
# Burp Repeater: hold the proof, mutate the identity field; test empty tokens.
```

## Verification & Testing

### Manual Checks
- Does the verifier pin `alg` to an explicit allow-list, rejecting HS256 when RS256 is expected?
- Can a session be created with a victim's identity field and the attacker's own valid token?
- Does an emptied/forged reset token still succeed?

### Automated Testing
```python
def test_alg_confusion_rejected():
    forged = forge_hs256(public_key_pem, sub="administrator")
    assert call_admin(forged).status_code in (401, 403)

def test_reset_requires_valid_token():
    r = post("/forgot-password", data={"temp-forgot-password-token": "", "username": "victim",
                                        "new-password-1": "x", "new-password-2": "x"})
    assert "reset" not in r.text.lower()   # fails closed without a valid token
```

### Security Scanning
Token/identity verification is mostly outside scanner reach; rely on the tests above plus secure-coding review
of the verification call sites.

## Related Best Practices

- [Web Application Penetration Testing Methodology](./web-application-penetration-testing-methodology.md)
- [Access Control & API Authorization Testing](./access-control-and-api-authorization-testing.md)
- [JWT Attacks and Defenses](../../02-Secure-Coding/best-practices) — JWT *defense* lives in the Secure Coding chapter (do not duplicate).

## Standards & Compliance

- **OWASP Top 10 2021:** A07:2021 Identification & Authentication Failures.
- **OWASP API Security Top 10 2023:** API2:2023 Broken Authentication.
- **CWE:** CWE-347 (Improper Verification of Cryptographic Signature), CWE-287 (Improper Authentication), CWE-640 (Weak Password Recovery).
- **RFC 8725:** JSON Web Token Best Current Practices (pin the algorithm).
- **OAuth 2.0 Security BCP:** prefer authorization-code + PKCE; verify tokens server-to-server.

## Further Reading

- [RFC 8725 — JWT Best Current Practices](https://datatracker.ietf.org/doc/html/rfc8725)
- [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
- [OWASP JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [PortSwigger: JWT attacks](https://portswigger.net/web-security/jwt) · [OAuth](https://portswigger.net/web-security/oauth)

## Case Studies

### Incident Example
Algorithm confusion has affected real JWT deployments that published a JWKS and used a verifier defaulting to
the header `alg`: a public key — safe to publish by design — became the symmetric secret, yielding admin
tokens. The fix is one line: pin `algorithms=["RS256"]`.

### Success Story
The "hold the proof, swap the identity" test (one Repeater edit) reliably surfaces the OAuth and password-reset
bugs in minutes — and the same requests become regression tests asserting the server now derives identity from
the verified credential alone.

## Reference Labs

The three trust-boundary tests above are grounded in these PortSwigger Web Security Academy labs (solved as
part of the contributor's portfolio):

- [JWT authentication bypass via algorithm confusion](https://portswigger.net/web-security/jwt/algorithm-confusion/lab-jwt-authentication-bypass-via-algorithm-confusion) — RS256→HS256 algorithm confusion (API2:2023)
- [Authentication bypass via OAuth implicit flow](https://portswigger.net/web-security/oauth/lab-oauth-authentication-bypass-via-oauth-implicit-flow) — identity decoupled from the access token (API2:2023)
- [Password reset broken logic](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic) — unvalidated reset token beside a client-supplied username (API2:2023)

## Tags

`authentication` `jwt` `algorithm-confusion` `oauth` `oidc` `password-reset` `trust-boundary` `identity` `owasp-api-top-10`

---

**Contributed by:** @roldao04
**Last Updated:** 2026-06-17
**Difficulty Level:** Advanced
**Impact:** High
