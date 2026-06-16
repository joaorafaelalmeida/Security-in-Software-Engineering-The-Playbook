# JWT Attacks and Defenses

> Verify the signature and pin the expected algorithm so a JSON Web Token cannot be forged through its own header.

## The Problem

A JSON Web Token (JWS form) is three Base64url segments joined by dots: `header.payload.signature`. Base64url is encoding, not encryption, so anyone holding a token can read the header and payload. The signature only proves those parts were not changed after signing.

Most JWT attacks exploit servers that trust attacker-controlled fields, especially the header `alg`, or that fail to verify the signature and expiry properly:

- If your server selects its verification routine from the token's own `alg` field, an attacker controls how (or whether) the token is checked. This opens both the `alg:none` and the RS256-to-HS256 confusion attacks below.
- If you decode and trust claims without verifying the signature, any tampered payload is accepted.
- If you do not enforce `exp`, an old or stolen token stays valid indefinitely.
- If you treat the payload as confidential, you leak every claim, because encoding is reversible by anyone with no key.

The consequences are direct: privilege escalation (raising `role` to `admin`), account impersonation (swapping `sub` or `email`), and disclosure of any secret mistakenly placed in a claim.

## The Solution

### Overview

Understand what each segment can and cannot do, then never let the token decide how it is verified.

| Segment | Contents | Who can read it | Who can forge it |
|---------|----------|-----------------|------------------|
| Header | `{"alg":"RS256","typ":"JWT"}` | Anyone | Anyone (structure is plain Base64url) |
| Payload | claims: `sub`, `role`, `iat`, `exp`, etc. | Anyone | Anyone (but a tampered payload breaks the signature) |
| Signature | HMAC or RSA signature over `header.payload` | Anyone (it is just bytes) | Only the holder of the signing key |

You can decode any segment with no key at all:

```bash
echo -n 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9' | tr '_-' '/+' | base64 -d
# {"alg":"HS256","typ":"JWT"}
```

The `tr '_-' '/+'` step converts Base64url back to standard Base64 before decoding (Base64url replaces `+/` with `-_` for URL safety).

### The Attacks

#### Attack 1: alg=none (Unsigned Token)

The `none` algorithm means "no signature". A permissive library that honors the token's own `alg` field will accept a token with `"alg":"none"` and an empty signature, treating it as valid without any cryptographic check.

How a forged token is built:

1. Take a legitimately issued token and decode its payload.
2. Edit any claim you want (for example raise `role` to `admin`, or swap `email`/`sub` to another user).
3. Build a new header `{"alg":"none","typ":"JWT"}` and Base64url-encode it.
4. Assemble the token as `HEADER.PAYLOAD.` with a trailing dot and an empty signature segment.

The server, trusting `alg:none`, performs no verification and accepts the forged claims. Note that the signature must be dropped entirely: simply editing the payload while keeping the original RS256/HS256 signature fails, because the signature was computed over the old payload and no longer matches. That is integrity working as intended.

Fix: never accept `none`. Pin the expected algorithm at verification time so a `none` header is rejected outright.

#### Attack 2: RS256 to HS256 Algorithm Confusion

RS256 is asymmetric: the server signs with a private key and verifies with the corresponding public key. HS256 is symmetric: the same secret both signs and verifies. The attack abuses a server that selects the verification routine from the token's `alg` field instead of pinning it.

The forgery:

1. Obtain the server's RSA public key (it is public by design; it may be exposed at an endpoint such as a JWKS URL or a key file).
2. Change the token header from `"alg":"RS256"` to `"alg":"HS256"`.
3. Edit the payload claims as desired.
4. Sign the token with HS256, using the full public key content as the HMAC secret.

If the verification code does the equivalent of `verify(token, publicKey)` without pinning the algorithm, then when the token claims HS256 the server uses the public key as a symmetric HMAC secret. Because the attacker also used that public key to sign, the HMAC matches and the forged token verifies. The whole attack rests on the public key being public: anyone can obtain it, so anyone can produce a matching HS256 signature.

Fix: pin the expected algorithm server-side (`algorithms=["RS256"]`) so an HS256 token is rejected before any key is applied. Do not trust the token's `alg` field to decide the verification method. Exposing the public key is not itself the root cause, but it makes this attack trivial, so avoid unnecessarily exposing keys.

#### Encoding Is Not Encryption

A signed JWT (JWS) provides integrity, not confidentiality. The payload is Base64url-encoded, which is fully reversible with no key, so every claim is world-readable to anyone who intercepts the token. Encryption, by contrast, requires a key to reverse. The signature does not hide the contents; it only makes tampering detectable. If the contents genuinely must be secret, use an encrypted token (JWE), and never rely on encoding to protect sensitive data.

### Defenses

- **Pin the algorithm.** Pass an explicit allow-list at verification time, for example `algorithms=["RS256"]` or `algorithms=["HS256"]`. This single control closes both `alg:none` and RS256-to-HS256 confusion, because the declared `alg` no longer decides how verification happens.
- **Always verify the signature.** Never decode-and-trust. A token whose signature does not match must be rejected.
- **Keep `exp` short and actually check it.** Use lifetimes of minutes to hours, not years, and refresh with refresh tokens. A valid signature is not the same as a currently usable token: an expired token with a perfect signature must still be rejected on the `exp` claim. Verification should fail closed when `exp` has passed.
- **Never put secrets in claims.** The payload is readable, so passwords, API keys, card numbers, or any secret in a claim leaks to anyone who holds the token.
- **Prefer asymmetric signing.** RS256 (or ES256) keeps the private signing key off the verifying services. If you use HS256, use a long, random secret (at least 32 bytes for SHA-256, per RFC 7518 section 3.2); short keys trigger warnings and weaken the HMAC.
- **Store tokens safely.** Prefer `HttpOnly`, `Secure`, `SameSite` cookies so client JavaScript cannot read the token. Tokens in `localStorage` are exposed to any XSS on the page, which turns a script injection into full account takeover.
- **Do not expose keys unnecessarily.** Public keys are meant to be public, but easy exposure lowers the bar for algorithm confusion. Restrict key endpoints to what is actually required.

## Code Examples

### Bad Practice (Vulnerable)

```python
# The library picks the algorithm from the token's own header.
# An HS256 token signed with the public key, or an alg:none token, both verify.
claims = jwt.verify(token, public_key)
user = claims["sub"]
```

**Why this is problematic:**

- The token controls the verification routine. An attacker sends `"alg":"HS256"` and signs with the public key (used as an HMAC secret), and the signature matches.
- The same trust-the-header behavior accepts `"alg":"none"` with an empty trailing-dot signature, skipping verification entirely.
- A forged `role` or `sub` claim is accepted, leading to privilege escalation or impersonation.

### Good Practice (Secure)

```python
# Pin the expected algorithm. An HS256 or none token is rejected before any key is applied.
claims = jwt.decode(token, key, algorithms=["RS256"])
user = claims["sub"]
```

**Why this works:**

- The allow-list, not the header, decides how the token is verified, so `alg:none` and RS256-to-HS256 confusion are both rejected.
- The signature is checked against the pinned algorithm; a tampered payload no longer matches.
- A standard library also enforces `exp` during `decode`, so an expired-but-validly-signed token fails closed.

## Common Pitfalls

1. **Trusting the header `alg`.** Calling `verify(token, key)` without an algorithm allow-list lets the attacker choose the verification method. Always pin.
2. **Editing the payload while keeping the old signature.** This fails by design; do not assume tampering is possible without also defeating the signature.
3. **Treating a valid signature as a valid session.** A signature can be perfect while `exp` has passed. Check expiry separately and fail closed.
4. **Putting secrets in claims.** The payload is world-readable. Encoding is not encryption.
5. **Using a weak HS256 secret.** Short secrets weaken the HMAC; use at least 32 bytes of randomness for SHA-256.
6. **Storing tokens in `localStorage`.** Any XSS then reads the token; prefer `HttpOnly`, `Secure`, `SameSite` cookies.

## When to Apply

- **Always:** When verifying any JWT used for authentication or authorization. Pinning the algorithm and verifying signature plus expiry is mandatory.
- **Recommended:** When choosing a signing scheme for a new service, prefer asymmetric signing (RS256 or ES256) so private keys stay off verifying services.
- **Consider:** When claims could ever be sensitive, use an encrypted token (JWE) instead of relying on a signed-but-readable JWS.

To practice both attacks hands-on (an unsigned `alg:none` token and an RS256-to-HS256 confusion using the server's published public key as the HMAC secret), OWASP Juice Shop is a convenient target.

## Standards & Compliance

- **OWASP Top 10:** A07:2021 Identification and Authentication Failures; A02:2021 Cryptographic Failures.
- **CWE:** CWE-347 Improper Verification of Cryptographic Signature.
- **RFC 7519:** JSON Web Token (JWT).
- **RFC 7518:** JSON Web Algorithms (JWA), including the minimum HMAC key length in section 3.2.

## Further Reading

- [RFC 7519 - JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519)
- [RFC 7518 - JSON Web Algorithms (JWA)](https://datatracker.ietf.org/doc/html/rfc7518)
- [OWASP JSON Web Token for Java Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [CWE-347: Improper Verification of Cryptographic Signature](https://cwe.mitre.org/data/definitions/347.html)

## Tags

`jwt` `authentication` `cryptography` `token-security` `owasp-top-10`

---

**Contributed by:** Henrique Teixeira
**Last Updated:** 2026-06-16
**Difficulty Level:** Intermediate
**Impact:** High
