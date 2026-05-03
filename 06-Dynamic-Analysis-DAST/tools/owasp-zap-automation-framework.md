# OWASP ZAP Automation Framework

> A YAML-based way to automate DAST scans with OWASP ZAP against running web applications.

## Overview

OWASP ZAP is a web application security testing tool. It can be used manually as an intercepting proxy, but it can also run automated scans in CI/CD pipelines or local Docker environments.

The ZAP Automation Framework lets teams define a scan in one YAML file. This makes DAST more repeatable because the same plan can be used by developers, security testers, and CI/CD jobs.

In an introductory Software Engineering Security course, this tool is useful because it shows the difference between SAST and DAST:

- SAST analyzes source code before the application runs.
- DAST tests a running application from the outside, mainly through HTTP requests and responses.

## Key Features

- **YAML automation plans:** define the target, authentication, scan jobs, and reports in one file.
- **Spidering:** discover pages, links, forms, and endpoints before testing.
- **Active scanning:** send attack payloads to discovered inputs and routes.
- **Authentication support:** scan areas that require login.
- **Report generation:** generate HTML and other report formats for review.
- **Docker support:** easy to run locally or in CI/CD.
- **CI/CD friendly:** can be used in scheduled jobs, pull request checks, or release gates.

## Use Cases

- Run a DAST scan against a local vulnerable app such as OWASP Juice Shop.
- Scan a staging environment before release.
- Generate security reports for developers.
- Validate whether a web application exposes weak headers, CORS issues, reflected input, or other runtime problems.
- Run deeper scheduled scans that are too slow for every pull request.

## Installation & Setup

### Prerequisites

- Docker
- Docker Compose
- A running web application target

For learning purposes, OWASP Juice Shop is a good target because it is deliberately vulnerable and designed for security training.

### Example Docker Compose Setup

```yaml
services:
  juice-shop:
    image: bkimminich/juice-shop:latest
    container_name: juice-shop
    ports:
      - "3000:3000"
    networks:
      - dast-network
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
        condition: service_healthy
    volumes:
      - ./zap-reports:/zap/wrk/:rw
    user: "0:0"
    command: zap.sh -cmd -autorun /zap/wrk/zap-plan.yaml
    networks:
      - dast-network

networks:
  dast-network:
    driver: bridge
```

Start the target:

```bash
docker compose up -d juice-shop
```

Run ZAP:

```bash
docker compose up zap-scanner
```

## Usage Examples

### Basic ZAP Plan

This plan discovers the application with the traditional spider, runs an active scan, and generates an HTML report.

```yaml
env:
  contexts:
  - name: "juice-shop"
    urls:
    - "http://juice-shop:3000"
    includePaths: []
    excludePaths: []
  parameters:
    failOnError: false
    failOnWarning: false
    progressToStdout: true

jobs:
- type: spider
  parameters:
    context: "juice-shop"
    url: "http://juice-shop:3000"
    maxDuration: 1

- type: activeScan
  parameters:
    context: "juice-shop"

- type: report
  parameters:
    template: traditional-html
    reportDir: /zap/wrk
    reportFile: zap-report.html
    reportTitle: "ZAP Automation Framework Report"
```

### Authenticated Scan Example

Many real vulnerabilities are behind login screens. The Automation Framework can define users and authentication logic.

```yaml
env:
  contexts:
  - name: "juice-shop"
    urls:
    - "http://juice-shop:3000"
    includePaths: []
    excludePaths:
    - "http://juice-shop:3000/rest/user/logout"
    authentication:
      method: "json"
      parameters:
        loginPageUrl: "http://juice-shop:3000/#/login"
        loginRequestUri: "http://juice-shop:3000/rest/user/login"
        loginRequestBody: '{"email":"{%username%}","password":"{%password%}"}'
      verification:
        method: "response"
        loggedOutRegex: "401 Unauthorized"
        loggedInRegex: ".*token.*"
    users:
    - name: "admin-user"
      credentials:
        username: "admin@juice-sh.op"
        password: "admin123"

jobs:
- type: spider
  parameters:
    context: "juice-shop"
    user: "admin-user"
    url: "http://juice-shop:3000"
    maxDuration: 1

- type: activeScan
  parameters:
    context: "juice-shop"
    user: "admin-user"
    maxRuleDurationInMins: 1
    maxScanDurationInMins: 2
  policyDefinition:
    defaultStrength: Low
    defaultThreshold: Medium

- type: report
  parameters:
    template: traditional-html
    reportDir: /zap/wrk
    reportFile: zap-report.html
    reportTitle: "ZAP Automation Framework Report"

- type: report
  parameters:
    template: risk-confidence-html
    reportDir: /zap/wrk
    reportFile: risk-report.html
    reportTitle: "ZAP Risk/Confidence Report"
```

## CI/CD Integration

DAST scans can be slow, so they are often better as scheduled jobs or release checks instead of blocking every small commit.

Example GitHub Actions workflow:

```yaml
name: Continuous DAST Pipeline

on:
  push:
    branches: [ "main" ]
  workflow_dispatch:

jobs:
  zap-dast-scan:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Start Target Application
        run: docker compose up -d juice-shop

      - name: Wait for Target
        run: |
          until curl -s http://localhost:3000 > /dev/null; do
            sleep 5
          done

      - name: Run ZAP Automation Framework
        run: docker compose up zap-scanner

      - name: Upload ZAP Report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: zap-report
          path: zap-reports/zap-report.html
```

## Pros and Cons

### Advantages

- Tests the application in a real running state.
- Finds runtime issues that SAST may miss, such as missing headers or exposed endpoints.
- Works well with Docker and CI/CD.
- YAML plans make scans repeatable.
- Good for learning because results are visible in reports.

### Limitations

- Slower than SAST.
- Needs a running and stable test environment.
- May miss routes if spidering is incomplete.
- Authentication can be fragile if login flows change.
- Findings still need human review because false positives can happen.

## Common Pitfalls

1. **Wrong target URL**
   - Inside Docker, ZAP should usually scan the service name, for example `http://juice-shop:3000`, not `http://localhost:3000`.

2. **Application not ready**
   - Add healthchecks or wait steps before starting ZAP.

3. **Running full scans too often**
   - Use fast scans for frequent feedback and deeper scans on a schedule.

4. **No authentication**
   - Without authentication, ZAP may only test public pages.

5. **Ignoring report triage**
   - A generated report is not enough. Findings should be reviewed, prioritized, and tracked.

## Alternatives

- **Burp Suite Enterprise:** commercial DAST platform with strong enterprise workflow support.
- **Nuclei:** template-based scanner useful for known exposures and misconfigurations.
- **Nikto:** older web server scanner, useful for simple server checks but less complete for modern apps.

## When to Use It

Use OWASP ZAP Automation Framework when:

- the application can run locally or in a test environment;
- the team wants repeatable DAST scans;
- the project needs visible security reports;
- runtime behaviour matters;
- the scan can be given enough time to crawl and test the application.

For a small student project, a good starting point is:

1. run SAST in pre-commit or pull requests;
2. run ZAP against the app in Docker;
3. review the ZAP report manually;
4. document which findings are true positives, false positives, or accepted risks.

## Resources

- [OWASP ZAP Automation Framework Documentation](https://www.zaproxy.org/docs/automate/automation-framework/)
- [OWASP ZAP Docker Documentation](https://www.zaproxy.org/docs/docker/)
- [OWASP Juice Shop Project](https://owasp.org/www-project-juice-shop/)
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

## License

Open source. OWASP ZAP is released under the Apache License 2.0.

## Tags

`dast` `owasp-zap` `automation-framework` `devsecops` `docker` `ci-cd`

---

**Contributed by:** João Varela  
**Last Updated:** 2026-05-03
