# Security Logging and Tamper-Resistant Audit Trails

> Record security-relevant activity as structured, protected events so attacks can be detected and investigated without exposing sensitive data.

## The Problem

When an account is compromised or a privileged feature is abused, responders need to know who performed an action, what was affected, when it happened, and whether it succeeded. Ordinary debug logs rarely provide this evidence consistently.

Poor logging creates additional risks:

- missing events prevent detection and investigation;
- excessive logs hide useful signals and increase cost;
- passwords, tokens, and personal data may be exposed;
- user-controlled text can forge entries through log injection;
- local files can be deleted by an attacker;
- alerts may be too noisy or have no assigned responder.

Logging is therefore not just writing messages. It requires useful events, safe content, protected storage, and actionable alerts.

## The Solution

### 1. Define Security Events

Create an inventory of the events the application must record. Prioritize:

- authentication successes and failures;
- account recovery, lockout, and multi-factor authentication changes;
- authorization failures;
- role, permission, and administrator changes;
- access to sensitive records;
- bulk exports and deletions;
- API key and credential lifecycle events, without the credential value;
- security configuration changes;
- rate-limit and validation failures that may indicate probing;
- failures in security controls or log delivery.

Each event should support an investigation question, alert, or compliance requirement. Record successful privileged actions as well as failures.

### 2. Use Structured Events

Prefer JSON or another structured format over manually assembled text:

```json
{
  "timestamp": "2026-06-13T14:22:31.482Z",
  "event_name": "authorization.denied",
  "severity": "medium",
  "service": "document-api",
  "actor": {
    "id": "usr_7f21",
    "type": "user"
  },
  "action": "document.read",
  "resource": {
    "type": "document",
    "id": "doc_91ac"
  },
  "outcome": "denied",
  "reason": "insufficient_role",
  "request_id": "req_01JX"
}
```

Use:

- UTC timestamps and synchronized clocks;
- stable, machine-readable event names;
- internal actor and resource identifiers;
- action, outcome, and reason fields;
- request or trace IDs for correlation;
- consistent severity levels.

Avoid placing important data only inside a free-form message because wording changes can break searches and alerts.

### 3. Keep Sensitive Data Out

Do not log:

- passwords or password reset values;
- session IDs, cookies, or authorization headers;
- access, refresh, identity, or API tokens;
- private keys and database connection strings;
- full payment details;
- unnecessary request bodies or personal data.

When correlation is required, use a masked value, internal identifier, or keyed hash instead of the original sensitive value. Redact data before it reaches the logging transport or storage system.

### 4. Prevent Log Injection

Request data and other external values are untrusted. In line-oriented logs, carriage returns and line feeds can create forged entries.

Use a structured logging API that encodes values separately. If plain-text logs are unavoidable:

- remove or encode CR and LF characters;
- validate expected formats;
- limit field length;
- escape values for the destination format;
- never build log lines through string concatenation.

### 5. Centralize and Protect Logs

Forward security events to a separate logging platform or SIEM. Local application files may disappear with a container or be deleted after a host compromise.

Protect the logging system by:

- encrypting data in transit and at rest;
- authenticating event producers;
- separating read, write, export, and administration permissions;
- preventing application identities from editing accepted records;
- using append-only or immutable retention when justified;
- monitoring ingestion gaps and collector failures;
- defining retention and secure deletion periods.

The goal is tamper resistance, not a claim that records are impossible to alter.

### 6. Build Actionable Alerts

Alert on meaningful patterns rather than every individual event. Examples include:

- one source attempting many accounts;
- one account failing authentication from multiple sources;
- successful login immediately after repeated failures;
- privilege escalation followed by a bulk export;
- repeated authorization failures across sensitive resources;
- audit logging being disabled or ingestion stopping unexpectedly.

Every alert needs an owner, severity, response target, investigation steps, and tuning process. Test the full path from generating an event to notifying the responder.

## Code Examples

### Bad Practice

```javascript
app.post("/login", async (req, res) => {
  const user = await findUser(req.body.email);

  if (!user || !(await verifyPassword(req.body.password, user.passwordHash))) {
    console.log(
      `Login failed: email=${req.body.email} password=${req.body.password}`
    );
    return res.status(401).send("Invalid credentials");
  }

  console.log(`Login successful for ${req.body.email}`);
  return res.send(createSession(user));
});
```

This exposes a password, accepts log-injection characters, and produces free-form messages without stable fields or request correlation.

### Good Practice

```javascript
import crypto from "node:crypto";

function emailCorrelationId(email) {
  return crypto
    .createHmac("sha256", process.env.LOG_CORRELATION_KEY)
    .update(email.trim().toLowerCase())
    .digest("hex");
}

app.post("/login", async (req, res) => {
  const email = String(req.body.email ?? "");
  const user = await findUser(email);
  const valid =
    user && (await verifyPassword(req.body.password, user.passwordHash));

  if (!valid) {
    securityLogger.info({
      event_name: "authentication.failed",
      actor_correlation_id: emailCorrelationId(email),
      source_ip: req.ip,
      action: "session.create",
      outcome: "denied",
      reason: "invalid_credentials",
      request_id: req.id
    });

    return res.status(401).send("Invalid credentials");
  }

  securityLogger.info({
    event_name: "authentication.succeeded",
    actor_id: user.id,
    source_ip: req.ip,
    action: "session.create",
    outcome: "success",
    request_id: req.id
  });

  return res.send(createSession(user));
});
```

The password and session value are excluded, events use stable fields, and failures can be correlated without storing the submitted email address. The logger must still be configured for structured output, automatic redaction, and protected transport.

## Common Pitfalls

1. **Logging only errors:** Successful administrative actions may be the most important events.
2. **Logging entire requests:** Bodies and headers often contain credentials or personal data.
3. **Keeping logs only locally:** Attackers may erase the evidence they create.
4. **Using free-form messages as a schema:** Small wording changes break detections.
5. **Ignoring logging failures:** Missing events and ingestion gaps require alerts.
6. **Alerting without ownership:** An alert without a responder and runbook is stored noise.
7. **Keeping logs forever:** Retention must balance investigations, privacy, obligations, and cost.

## Verification Checklist

- Trigger every required event and confirm its fields.
- Search logging destinations for test passwords, tokens, cookies, and authorization headers.
- Submit CR, LF, quotes, and delimiters in logged inputs.
- Confirm request IDs connect events across services.
- Verify application identities cannot edit or delete accepted audit records.
- Interrupt the collector and confirm buffering, health monitoring, and alerts.
- Generate a detection scenario and verify the correct responder is notified.
- Test retention, export, and restoration procedures.

Example automated test:

```javascript
test("failed login does not log credentials", async () => {
  await request(app).post("/login").send({
    email: "attacker@example.test\r\nforged=true",
    password: "NeverLogThis!"
  });

  const event = securityLogSink.lastEvent();
  const serialized = JSON.stringify(event);

  expect(event.event_name).toBe("authentication.failed");
  expect(serialized).not.toContain("NeverLogThis!");
  expect(serialized).not.toContain("\r");
  expect(serialized).not.toContain("\n");
});
```

## Standards and References

- **OWASP Top 10:2025 A09:** Security Logging and Alerting Failures.
- **OWASP ASVS 5.0 V16:** Security logging, event protection, and error handling.
- **CWE-117:** Improper Output Neutralization for Logs.
- **NIST SP 800-92:** Enterprise log management guidance.
- **NIST SP 800-53 Rev. 5:** Audit and Accountability controls.
- **PCI DSS v4.0.1 Requirement 10:** Logging and monitoring access to in-scope systems and cardholder data.

Further reading:

- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [OWASP Top 10:2025 A09](https://owasp.org/Top10/2025/A09_2025-Security_Logging_and_Alerting_Failures/)
- [NIST SP 800-92](https://csrc.nist.gov/pubs/sp/800/92/final)
- [CWE-117](https://cwe.mitre.org/data/definitions/117.html)

## Tags

`security-logging` `audit-trails` `monitoring` `incident-response` `log-injection`

---

**Contributed by:** [Diogo Fernandes](https://github.com/diogux)
**Last Updated:** 2026-06-13
**Difficulty Level:** Intermediate
**Impact:** High
