# Authenticated OWASP ZAP DAST Scans in CI

> Run repeatable, logged-in OWASP ZAP scans against a containerized target in CI so you exercise the real attack surface behind login, not just public pages.

## The Problem

SAST reads source code and never sees how an app behaves once it is deployed. Missing security headers, misconfigurations, and runtime behavior only show up when something sends real HTTP requests at a running instance. Dynamic Application Security Testing (DAST) covers that gap, but most DAST runs fall short in two ways:

- **They run unauthenticated.** An anonymous scan only reaches public pages, so it mostly reports header and configuration issues: Content Security Policy header not set, permissive CORS or cross-domain misconfiguration, timestamp disclosure. Those are real, but they are the shallow surface. Account actions, admin functions, and data access all live behind the login. Skip authentication and you will conclude a known-vulnerable app is "mostly fine," which is misleading.
- **They are run by hand.** Clicking through the ZAP desktop UI is not repeatable and cannot run in CI. The scan, its scope, and its outputs need to be scripted to be trustworthy over time.

## The Solution

### Overview

Use the OWASP ZAP Automation Framework to define the scan as a YAML plan (spider, then active scan, then report), run the target and ZAP as two services on one Docker Compose network, authenticate the scan as a real user so it crawls and attacks logged-in routes, and wire the whole thing into CI as a scheduled job that always preserves its report. Authenticated, automated, and runtime: the combination is what surfaces the higher-impact findings.

### Implementation

#### Step 1: Run the target and ZAP on one Compose network

Run the target application and ZAP (headless) as two services on the same network. ZAP reaches the target by its service name, and a shared volume lets it read the automation plan and write the report back to your host.

```yaml
services:
  juice-shop:                          # the target app (swap for your own)
    image: bkimminich/juice-shop:latest
    container_name: juice-shop
    ports:
      - "3000:3000"
    networks:
      - dast-network
    # Healthcheck so ZAP does not start before the target is ready
    healthcheck:
      test: [ "CMD", "/nodejs/bin/node", "-e", "fetch('http://localhost:3000/').then(r => process.exit(r.ok ? 0 : 1)).catch(() => process.exit(1))" ]
      interval: 10s
      timeout: 5s
      retries: 12

  zap-scanner:
    image: zaproxy/zap-stable:latest
    container_name: zap-scanner
    depends_on:
      juice-shop:
        condition: service_healthy     # wait for the healthcheck to pass
    volumes:
      - ./zap-reports:/zap/wrk/:rw     # plan in, report out
    user: "0:0"                        # root, so ZAP can write the report file
    command: zap.sh -cmd -autorun /zap/wrk/zap-plan.yaml
    networks:
      - dast-network

networks:
  dast-network:
    driver: bridge
```

The healthcheck is load-bearing. Without `condition: service_healthy`, ZAP starts the moment the target container exists, before the app inside it is actually listening. ZAP then hits connection errors, finds nothing, and produces a basically empty report. The healthcheck plus `depends_on` gates the scanner until the target answers requests. Pick a healthcheck command that exists inside the target image (the example uses the Node binary that ships in Juice Shop; adjust for your stack).

#### Step 2: Define the automation plan (spider, scan, report)

The plan is a YAML file with an `env` block (contexts, auth, users) followed by jobs that run in order. The core sequence is spider, then active scan, then report.

```yaml
env:
  contexts:
  - name: "juice-shop"
    urls:
    - "http://juice-shop:3000"
    includePaths: []

# Step 1: Spider, crawl the app to discover endpoints
- type: spider
  parameters:
    context: "juice-shop"
    url: "http://juice-shop:3000"
    maxDuration: 1          # minutes

# Step 2: Active scan, inject attack payloads into what the spider found
- type: activeScan
  parameters:
    context: "juice-shop"

# Step 3: Report
- type: report
  parameters:
    template: traditional-html
    reportDir: /zap/wrk
    reportFile: zap-report.html
    reportTitle: "ZAP Automation Framework Report"
```

#### Step 3: Add JSON-based authentication

To reach pages behind login, define an authentication strategy and a user inside the context. For an app whose login endpoint accepts a JSON body (many SPAs do), use `method: "json"`:

```yaml
env:
  contexts:
  - name: "juice-shop"
    urls:
    - "http://juice-shop:3000"
    includePaths: []
    # Never scan or click the logout path, or ZAP logs itself out mid-scan
    excludePaths:
    - "http://juice-shop:3000/rest/user/logout"
    authentication:
      method: "json"
      parameters:
        loginPageUrl: "http://juice-shop:3000/#/login"
        loginRequestUri: "http://juice-shop:3000/rest/user/login"
        loginRequestBody: '{"email":"{%username%}","password":"{%password%}"}'
      # How ZAP decides whether it is logged in, by inspecting the response
      verification:
        method: "response"
        loggedOutRegex: "401 Unauthorized"
        loggedInRegex: ".*token.*"
    users:
    - name: "admin-user"
      credentials:
        username: "admin@juice-sh.op"
        password: "admin123"
```

Three details make or break this:

- **Login endpoint and body.** `loginRequestUri` is where the credentials are POSTed; `loginRequestBody` is the JSON template, with `{%username%}` and `{%password%}` substituted from the user's credentials.
- **Verification regex.** ZAP needs to know whether a response means logged in or logged out. A token-bearing login response is a reliable signal, so `loggedInRegex: ".*token.*"` matches the auth token in the body. Pair it with a `loggedOutRegex` (here `401 Unauthorized`) so ZAP can detect a dropped session and re-authenticate.
- **Exclude the logout path.** Add the logout URL to `excludePaths` so the spider and scanner never trigger it and kill their own session.

#### Step 4: Assign the user to the jobs

Defining a user is not enough. The authentication config just sits there unless you tell each job to run as that user. Add `user: "admin-user"` to both the spider and the active scan:

```yaml
- type: spider
  parameters:
    context: "juice-shop"
    user: "admin-user"        # crawl as the authenticated user
    url: "http://juice-shop:3000"
    maxDuration: 1

- type: activeScan
  parameters:
    context: "juice-shop"
    user: "admin-user"        # attack as the authenticated user
```

This is the single most common mistake: auth is configured, the scan runs green, but it still only saw public pages because no job referenced the user.

#### Step 5: Tune the active scan to your time budget

The active scan dominates the runtime, so tune it deliberately:

```yaml
- type: activeScan
  parameters:
    context: "juice-shop"
    user: "admin-user"
    policyDefinition:
      defaultStrength: Low      # fewer payloads per parameter, faster scan
      defaultThreshold: Medium  # only report findings ZAP is reasonably confident about
    maxRuleDurationInMins: 1    # cap per-rule time
    maxScanDurationInMins: 2    # cap total scan time
```

`defaultStrength: Low` trades depth for speed and may miss edge-case vulnerabilities; raise it when you have time. `defaultThreshold: Medium` reduces noisy low-confidence findings. The two duration caps stop a scan from running indefinitely, which matters in CI.

#### Step 6: Emit multiple report formats

A single report job can be repeated with different templates to produce more than one view:

```yaml
- type: report
  parameters:
    template: traditional-html       # detailed, human-readable
    reportDir: /zap/wrk
    reportFile: zap-report.html

- type: report
  parameters:
    template: risk-confidence-html   # severity x confidence matrix
    reportDir: /zap/wrk
    reportFile: zap-report-risk.html
```

The risk-versus-confidence matrix helps with triage: a High-severity finding with Low confidence is likely a false positive, while a Medium-severity finding with High confidence is worth fixing now.

#### Step 7: Wire it into CI (GitHub Actions)

Run the scan on a schedule (and optionally on push). The job starts the target detached, polls until it answers, runs ZAP against it, and uploads the report.

```yaml
name: DAST Scan
on:
  push:
    branches: [ main ]
  schedule:
    - cron: "0 2 * * *"   # nightly at 02:00 UTC

jobs:
  zap-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start target
        run: docker compose up -d juice-shop

      - name: Wait until target is healthy
        run: |
          for i in $(seq 1 30); do
            if curl -sf http://localhost:3000/ > /dev/null; then exit 0; fi
            sleep 5
          done
          echo "Target never became healthy" && exit 1

      - name: Run ZAP scan
        run: docker compose up --exit-code-from zap-scanner zap-scanner

      - name: Upload report
        if: always()                 # keep the report even if the scan fails the build
        uses: actions/upload-artifact@v4
        with:
          name: zap-report
          path: zap-reports/zap-report.html
          retention-days: 7
```

`if: always()` is essential: GitHub destroys the runner VM when the job ends, so if the scan finds issues and fails the step, the report is still uploaded and not lost.

## Common Pitfalls

1. **Missing healthcheck, empty report.** ZAP starts before the target is listening, gets connection errors, and reports nothing. Gate ZAP on `condition: service_healthy`.
2. **Auth configured but not assigned to jobs.** The context has authentication and a user, but the spider and active scan have no `user:` parameter, so they scan only public pages. Add `user: "<name>"` to every job that should be authenticated.
3. **Logging yourself out.** If the logout path is not in `excludePaths`, the spider or scanner can hit it and drop the session, silently degrading an authenticated scan to an unauthenticated one.
4. **Scan time versus coverage.** Low strength and tight duration caps finish fast but miss findings; high strength is thorough but slow. Choose deliberately for the context (quick check versus deep nightly run).
5. **Report lost in CI.** Without an artifact upload (and `if: always()`), the report dies with the runner. Always upload it.

## When to Apply

- **Always:** When testing an application that has a login and meaningful functionality behind it. An unauthenticated scan of such an app gives a false sense of safety.
- **Recommended:** As a scheduled (nightly) job rather than on every pull request. DAST is much slower than SAST: it has to boot the target, wait for health, spider, and attack, which takes minutes at minimum. Keep fast SAST on each PR and let the heavier DAST run on a schedule.
- **Consider:** Running on push to key branches when the application is small and the scan stays within a tight time budget.

## Standards & Compliance

- **OWASP Top 10:** A05:2021 Security Misconfiguration is the category most unauthenticated ZAP findings fall under, including missing Content Security Policy headers and permissive CORS or cross-domain misconfiguration. Authenticated scans extend coverage into access-control and injection categories that only appear behind login.

## Further Reading

- [OWASP ZAP Automation Framework Documentation](https://www.zaproxy.org/docs/automate/automation-framework/)
- [OWASP ZAP Automation Framework Jobs Reference (spider, activeScan, report, authentication)](https://www.zaproxy.org/docs/desktop/addons/automation-framework/)
- [OWASP Top 10 A05:2021 Security Misconfiguration](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/)

## Tags

`dast` `owasp-zap` `ci-cd` `authentication` `docker` `automation`

---

**Contributed by:** Henrique Teixeira
**Last Updated:** 2026-06-16
**Difficulty Level:** Intermediate
**Impact:** High
