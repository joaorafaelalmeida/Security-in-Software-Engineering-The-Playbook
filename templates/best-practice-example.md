# Best Practice Title

> One-line description of the practice and its security benefit

## The Problem

Describe the security challenge, vulnerability, or risk that this best practice addresses:

- What can go wrong if this practice is not followed?
- What are the potential consequences?
- Real-world examples or incidents (if available)

## The Solution

### Overview

A clear explanation of the best practice and how it mitigates the problem.

### Implementation

Step-by-step guidance on how to implement this practice:

#### Step 1: [Action]
Description and explanation of the first step.

```language
// Code example if applicable
example_code_here();
```

#### Step 2: [Action]
Description and explanation of the second step.

#### Step 3: [Action]
Description and explanation of the third step.

## Code Examples

### Bad Practice (Vulnerable)

```language
// Example of what NOT to do
function vulnerableExample() {
    // Insecure code demonstrating the anti-pattern
    const password = request.params.password;
    const query = "SELECT * FROM users WHERE password = '" + password + "'";
    // SQL Injection vulnerability
}
```

**Why this is problematic:**
- Explanation of the security issue
- Potential attack vectors
- Impact if exploited

### Good Practice (Secure)

```language
// Example of the correct, secure approach
function secureExample() {
    // Secure code following best practices
    const password = request.params.password;
    const query = "SELECT * FROM users WHERE password = ?";
    db.execute(query, [password]); // Parameterized query
}
```

**Why this works:**
- Explanation of how this mitigates the risk
- Security mechanisms involved
- Additional benefits

## Benefits

- **Security Benefit 1:** Explanation
- **Security Benefit 2:** Explanation
- **Additional Benefit:** Performance, maintainability, etc.

## Common Pitfalls

Things to watch out for when implementing this practice:

1. **Pitfall 1:** Description and how to avoid it
2. **Pitfall 2:** Description and how to avoid it
3. **Pitfall 3:** Description and how to avoid it

## When to Apply

- **Always:** Situations where this practice is mandatory
- **Recommended:** Contexts where it's highly beneficial
- **Consider:** Scenarios where it may be optional but still valuable

## Framework/Language-Specific Guidance

### Python
```python
# Python-specific implementation
```

### JavaScript/Node.js
```javascript
// JavaScript-specific implementation
```

### Java
```java
// Java-specific implementation
```

(Include languages relevant to the practice)

## Verification & Testing

How to verify this practice is correctly implemented:

### Manual Checks
- Check 1: How to verify
- Check 2: How to verify

### Automated Testing
```language
// Example test case
test('should properly sanitize input', () => {
    // Test implementation
});
```

### Security Scanning
Tools that can detect violations of this practice:
- Tool 1
- Tool 2

## Related Best Practices

- [Related Practice 1](#) - Brief explanation of connection
- [Related Practice 2](#) - Brief explanation of connection
- [Related Practice 3](#) - Brief explanation of connection

## Standards & Compliance

Which security standards or frameworks recommend this practice:

- **OWASP Top 10:** Relevant category (e.g., A03:2021 - Injection)
- **CWE:** Relevant weakness ID (e.g., CWE-89: SQL Injection)
- **NIST:** Relevant guidelines
- **PCI DSS / GDPR / etc.:** Relevant compliance requirements

## Further Reading

- [Official Documentation](https://example.com/docs)
- [Security Guide](https://example.com/security-guide)
- [Research Paper](https://example.com/paper)
- [OWASP Reference](https://owasp.org/reference)

## Case Studies

Real-world examples where following/ignoring this practice had significant impact:

### Incident Example
Brief description of a security incident that could have been prevented by this practice.

### Success Story
Example of how implementing this practice improved security posture.

## Tags

`secure-coding` `input-validation` `injection-prevention` `owasp-top-10`

---

**Contributed by:** [Your Name]
**Last Updated:** YYYY-MM-DD
**Difficulty Level:** Beginner / Intermediate / Advanced
**Impact:** High / Medium / Low
