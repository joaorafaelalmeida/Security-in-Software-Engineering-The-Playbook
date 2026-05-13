# Input Validation and Sanitization

> Reject or neutralize untrusted input at every system boundary before it can influence business logic, queries, or output.

## The Problem

Every piece of data that enters a system from outside HTTP parameters, headers, file uploads, environment variables, inter-service messages can be a potential attack vector. When applications trust that input without verifying it, attackers can craft payloads that:

- inject SQL or NoSQL commands into database queries;
- embed scripts that execute in other users' browsers (XSS);
- traverse the filesystem beyond the intended directory (path traversal);
- trigger unexpected application behaviour through oversized or malformed values.

This is a real concern, especially because it has affected real companies. In 2017, the Equifax breach exposed personal information of about 147 million people. The attack vector was an unpatched Apache Struts vulnerability, CVE-2017-5638, in the online dispute portal. That vulnerability allowed remote code execution through crafted HTTP headers. OWASP consistently recognizes input-related failures as critical vulnerabilities: injection ranks as A03:2021 in the current OWASP Top 10, while broken access control (which path traversal falls under) holds the top spot at A01:2021.

The underlying cause is almost always the same: the application blindly trusted what it received instead of checking whether it was safe to receive and use.

## The Solution

### Overview

Input handling has two distinct responsibilities:

- **Validation** — decide whether the input is acceptable. If it is not, reject it early and return a clear error.
- **Sanitization** — transform acceptable input into a safe form before using it in a sensitive context (query, template, file path, shell command).
These are complementary, not interchangeable. Validation alone does not protect every output context, and transforming input too early can mask bugs that validation would surface immediately.

### Core Principle: Trust Nothing from Outside the Process

Treat any data that originates outside your application as untrusted. This includes HTTP request parameters, headers, cookies, uploaded files, URL segments, third-party API responses and even environment variables in some cases. 

### Implementation

#### Step 1: Validate at the Entry Point

Validate as close to the boundary as possible, ideally in a dedicated layer (middleware, a request schema, a DTO) before the value reaches business logic.

For each input field, define:
- **Type** — what kind of value is expected (string, integer, boolean, UUID, date, etc.). Rejecting a string where an integer is expected is one of the simplest and most effective checks you can make.
- **Required vs optional** — reject requests where required fields are missing rather than defaulting silently. Accepting missing fields and filling in defaults can mask bugs and open unexpected code paths.

Use an allowlist (define what is permitted) rather than a denylist (block known bad values) since denylists are inherently incomplete.

**Python — using Pydantic for schema validation:**

Pydantic lets you declare the shape and constraints of your input as a class. When a request comes in, you pass it to that class and it either returns a validated object or raises a `ValidationError`, with no manual checking needed.

```python
from typing import Annotated

from pydantic import BaseModel, EmailStr, Field, StringConstraints

class RegistrationRequest(BaseModel):
    username: Annotated[
        str,
        StringConstraints(min_length=3, max_length=32, pattern=r"^[a-zA-Z0-9_-]+$")
    ]
    age: Annotated[int, Field(ge=0, le=150, strict=True)]
    email: EmailStr
```

**Java Spring Boot — DTO with `@Valid` in the controller:**

In Spring Boot, you define the validation rules as annotations directly on the DTO fields. Adding `@Valid` to the controller method parameter tells Spring to run those checks automatically before your code executes. If anything fails, Spring returns a `400 Bad Request` with field-level errors without you writing any validation logic manually.

```java
// DTO
import jakarta.validation.constraints.*;

public class RegistrationRequest {
    @NotBlank
    @Size(min = 3, max = 32)
    @Pattern(regexp = "^[a-zA-Z0-9_-]+$")
    private String username;

    @Min(0) @Max(150)
    private int age;

    @Email
    @NotBlank
    private String email;

    // getters and setters
}

// Controller
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping("/register")
    public ResponseEntity<String> register(@Valid @RequestBody RegistrationRequest request) {
        // If validation fails, Spring automatically returns 400 Bad Request
        // with field-level error details — no manual checking needed here
        userService.register(request);
        return ResponseEntity.ok("Registered successfully");
    }
}
```

Spring Boot triggers validation automatically when `@Valid` is present. If any constraint is violated, it throws a `MethodArgumentNotValidException` and returns a `400 Bad Request` before your business logic even runs. You can customize the error response globally with a `@ControllerAdvice`.

#### Step 2: Reject Early, Fail Clearly

Return an error as soon as validation fails. Do not attempt to silently fix bad input (e.g., truncating a string that is too long or stripping characters to force a match). Silent fixes hide bugs, complicate debugging, and can be abused.

Return enough information for a legitimate client to correct the request, but not so much that you expose internal structure to an attacker.

```python
# Bad — silent fix
def get_user(user_id: str):
    user_id = user_id[:36]  # truncate to UUID length and hope for the best
    return db.query(user_id)

# Good — explicit rejection
def get_user(user_id: str):
    if not re.fullmatch(r'[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}', user_id):
        raise ValueError("Invalid user ID format")
    return db.query(user_id)
```

In the bad example, the code silently trims the input to 36 characters and passes it to the database anyway. An attacker sending a malformed or malicious ID would not be stopped — the code just quietly adjusts the input and continues. In the good example, the code checks that the value looks exactly like a UUID before doing anything with it. If it does not match, it raises an error immediately and the database is never touched.



#### Step 3: Sanitize for the Output Context

Even validated input can be dangerous when placed in a context it was not designed for. A username that is perfectly valid for storage can still contain characters that break an HTML template or a shell command.

Apply context-specific output encoding, parameterization, or escaping as late as possible, at the point of use, not at the point of entry.

| Output context | Required protection |
|---|---|
| HTML body | Context-specific output encoding (`&`, `<`, `>`, `"`, `'`) |
| HTML attribute | Context-specific output encoding, quote the attribute |
| SQL query | Use parameterized queries — never concatenate |
| Shell command | Use structured APIs (subprocess with list args, not shell=True) |
| File path | Resolve and verify the canonical path stays within the allowed directory |
| JSON response | Use a serializer — never build JSON by string concatenation |

**HTML output — Python (Jinja2 auto-escaping):**

```python
# Bad — raw string interpolation
html = f"<p>Welcome, {username}!</p>"

# Good — let the template engine escape
from jinja2 import Environment
env = Environment(autoescape=True)
template = env.from_string("<p>Welcome, {{ username }}!</p>")
html = template.render(username=username)
```

In the bad example, if `username` contains something like `<script>alert(1)</script>`, that script tag lands directly in the HTML and executes in the user's browser. In the good example, Jinja2 with `autoescape=True` converts those characters into harmless HTML entities, so the browser displays them as text instead of running them.

**SQL — parameterized queries:**

```python
# Bad — string concatenation leads to SQL injection
query = f"SELECT * FROM users WHERE username = '{username}'"
cursor.execute(query)

# Good — parameterized
cursor.execute("SELECT * FROM users WHERE username = %s", (username,))
```

In the bad example, an attacker can send a username like `' OR '1'='1` and break out of the intended query, potentially returning every row in the table. In the good example, the value is passed separately from the query and the database treats it purely as data, never as SQL — no matter what it contains.

**File path — prevent path traversal:**

```python
import os

UPLOAD_DIR = os.path.realpath("/var/app/uploads")

def save_file(filename: str, content: bytes):
    # Resolve the full path and verify it stays inside UPLOAD_DIR
    target = os.path.realpath(os.path.join(UPLOAD_DIR, filename))
    if os.path.commonpath([UPLOAD_DIR, target]) != UPLOAD_DIR:
        raise ValueError("Path traversal attempt detected")
    with open(target, "wb") as f:
        f.write(content)
```

Without the check, a filename like `../../etc/passwd` would resolve to a path outside the upload directory, letting an attacker read or overwrite sensitive system files. `os.path.realpath` resolves all `..` sequences and symbolic links to the true absolute path, and `os.path.commonpath` then ensures the result is still inside the allowed directory before the file is opened. File uploads should also use server-generated filenames, size limits, content validation, and malware scanning where feasible.

**Shell commands — avoid shell=True:**

```python
import subprocess

# Bad — shell=True with user input is command injection
subprocess.run(f"convert {filename} output.png", shell=True)

# Good — pass arguments as a list and use -- where the command supports it
subprocess.run(["convert", "--", filename, "output.png"], check=True)
```

With `shell=True`, the entire string is passed to the shell interpreter. An attacker who controls `filename` can inject additional commands, for example `photo.jpg; rm -rf /`. Passing arguments as a list bypasses the shell entirely, so each element is treated as one argument. However, a user-controlled filename that starts with `-` may still be interpreted as a command option. Where the command supports it, add `--` before the filename to mark the end of options.

## Common Pitfalls

1. **Validating only on the client side.** Client-side validation improves UX but is trivially bypassed. Always re-validate on the server.

2. **Using denylists instead of allowlists.** Blocking known-bad patterns (e.g., `<script>`) is fragile, as there are always encoding tricks and edge cases that bypass them. Define what is allowed instead.

3. **Validating at entry but not protecting the point of use.** A value stored safely in the database can still cause XSS if rendered without context-specific output encoding, or SQL injection if interpolated into a dynamic query later.

4. **Over-sanitizing at entry.** Stripping characters on the way in (e.g., removing `<` from all input) corrupts legitimate data and gives a false sense of security. Encode at the point of use instead.

5. **Forgetting indirect input sources.** HTTP headers (`User-Agent`, `Referer`, `X-Forwarded-For`), cookie values, and environment variables are all attacker-controlled in some scenarios. Apply the same validation discipline.

6. **Logging raw input before validation.** If validation fails, log a sanitized or truncated version of the input, not the raw value. Logging attacker-controlled strings can lead to log injection.

## When to Apply

- **Always:** Any value that originates outside the current process must be validated before use and protected appropriately before being placed in a sensitive context.
- **Always:** File uploads. Use server-generated filenames, size limits, content validation, and malware scanning where feasible.
- **Recommended:** Validate inter-service messages even on internal networks. Assume breach.
- **Consider:** Re-validating data read from the database before placing it in a new output context (e.g., user-supplied content rendered in a different service).

## Verification & Testing

### Manual Checks

- Trace every external input through the codebase to confirm it hits a validation layer before reaching business logic.
- Confirm that SQL queries use parameterized statements or an ORM that prevents interpolation.
- Confirm that HTML templates use auto-escaping and that raw/unsafe output helpers are not used with user-supplied data.

### Automated Testing

```python
# Test that invalid input is rejected
def test_rejects_oversized_username():
    with pytest.raises(ValidationError):
        RegistrationRequest(username="a" * 100, age=25, email="x@example.com")

# Test that SQL injection payloads do not alter query behaviour
def test_sql_injection_payload_returns_no_results(db):
    result = get_user_by_username("' OR '1'='1")
    assert result is None
```

### Security Scanning

- **Semgrep** — rules for detecting string concatenation in SQL queries, use of `shell=True`, missing output encoding.
- **Bandit** (Python) — flags `subprocess` calls with `shell=True` and SQL string interpolation.
- **ESLint security plugin** (JavaScript) — detects `eval`, unsafe regex, and injection-prone patterns.
- **OWASP ZAP / Burp Suite** — active scanning for injection and XSS in running applications.

## Related Best Practices

- [SQL Injection Prevention](#) — deeper focus on parameterized queries and ORM safe usage
- [XSS Prevention](#) — output encoding and Content Security Policy
- [Secure File Upload Handling](#) — validating file content, not just extension and MIME type

## Standards & Compliance

- **OWASP Top 10:** A03:2021 — Injection, A01:2021 — Broken Access Control (path traversal)
- **CWE:** CWE-20 (Improper Input Validation), CWE-89 (SQL Injection), CWE-79 (XSS), CWE-22 (Path Traversal)
- **NIST SP 800-53:** SI-10 (Information Input Validation)
- **PCI DSS:** Requirement 6.2.4 — software engineering practices to prevent common vulnerabilities including injection

## Further Reading

- [OWASP Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [CWE-20: Improper Input Validation](https://cwe.mitre.org/data/definitions/20.html)

## Tags

`secure-coding` `input-validation` `sanitization` `injection-prevention` `xss` `sql-injection` `path-traversal` `owasp-top-10`

---

**Contributed by:** Afonso Ferreira
**Last Updated:** 2026-05-10
**Difficulty Level:** Beginner / Intermediate 
**Impact:** High
