# SonarQube

> A self-hosted platform that combines static security analysis with code-quality and maintainability metrics, tracked over time with a "Quality Gate" you can fail builds on.

## Overview

SonarQube is a continuous code-inspection platform. Beyond pure SAST, it unifies **security vulnerabilities, security hotspots, bugs, code smells, coverage, and duplication** into a single dashboard with historical trends. Its signature feature is the **Quality Gate**: a configurable pass/fail threshold (e.g. "no new vulnerabilities and no new code below 80% coverage") that integrates with CI to block merges.

It is widely adopted in enterprises that want one platform spanning security and maintainability, with role-based dashboards, project portfolios, and long-term trend tracking. The hosted equivalent is **SonarCloud** (now SonarQube Cloud).

## Key Features

- **Security + quality in one place:** Vulnerabilities and *security hotspots* (code needing manual security review) alongside bugs and code smells.
- **Quality Gates:** Enforceable pass/fail conditions, with a strong "Clean as You Code" model focused on *new* code.
- **Taint analysis:** Source-to-sink vulnerability detection (injection, XSS) in commercial editions.
- **30+ languages** with deep ecosystem and CI integrations.
- **Trends and dashboards:** Historical metrics, portfolio views, and per-developer feedback.
- **IDE feedback via SonarLint:** Surfaces issues in the editor before commit.

## Use Cases

- Organizations wanting a single, persistent dashboard for security *and* maintainability.
- Enforcing a "no new issues on changed code" policy through Quality Gates in CI.
- Tracking security and tech-debt trends across many projects over time.
- Teams that value manual **security hotspot** review as part of their workflow.

## Installation & Setup

### Prerequisites
- Java runtime and a supported database (PostgreSQL recommended) for self-hosting, or use Docker.
- A CI scanner (SonarScanner CLI, Maven/Gradle plugin, or CI-native integration).

### Installation

```bash
# Run the SonarQube server locally with Docker
docker run -d --name sonarqube -p 9000:9000 sonarqube:community

# Then analyze a project with the scanner CLI
docker run --rm -e SONAR_HOST_URL="http://localhost:9000" \
  -e SONAR_TOKEN="$SONAR_TOKEN" \
  -v "$PWD:/usr/src" sonarsource/sonar-scanner-cli
```

## Usage Examples

### Basic Usage

```bash
# Analyze with the scanner CLI (reads sonar-project.properties)
sonar-scanner \
  -Dsonar.projectKey=my-app \
  -Dsonar.sources=src \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.token=$SONAR_TOKEN
```

### Configuration Example

```properties
# sonar-project.properties
sonar.projectKey=my-app
sonar.sources=src
sonar.tests=tests
sonar.python.coverage.reportPaths=coverage.xml
sonar.qualitygate.wait=true   # block CI until the Quality Gate result is known
```

### Quality Gate in CI

```yaml
# GitHub Actions
- uses: SonarSource/sonarqube-scan-action@v3
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
# The build fails automatically if the Quality Gate fails (sonar.qualitygate.wait=true)
```

## Pros and Cons

### Advantages ✅
- One platform for security, bugs, and maintainability with rich history.
- Quality Gates give a clear, enforceable merge criterion.
- "Clean as You Code" focuses effort on new code, aiding adoption on legacy projects.
- Strong IDE (SonarLint), CI, and PR-decoration integrations.

### Limitations ⚠️
- Self-hosting requires infrastructure (server + database) and upkeep.
- Deepest **taint analysis** and many languages are gated behind paid editions.
- Can surface a lot of non-security "code smells" that dilute security focus if not tuned.
- Heavier than a single-purpose scanner for small projects.

## Integration

```yaml
# Typical placement: PR analysis with Quality Gate as the merge gate
on:
  pull_request:
jobs:
  sonar:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }   # full history for accurate new-code detection
      - uses: SonarSource/sonarqube-scan-action@v3
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
```

## Alternatives

- **Semgrep** — lighter, faster, easier custom rules; no quality/maintainability platform.
- **CodeQL** — deeper taint analysis and variant analysis, but no quality dashboard.
- **Snyk** — combines SAST (Snyk Code) with SCA and container scanning.

## Resources

- [Official Documentation](https://docs.sonarsource.com/sonarqube/latest/)
- [SonarQube Community Edition](https://www.sonarsource.com/products/sonarqube/downloads/)
- [SonarSource on GitHub](https://github.com/SonarSource/sonarqube)
- [Clean as You Code](https://www.sonarsource.com/solutions/clean-as-you-code/)

## License

**Community Edition** is open source (LGPL v3). Developer, Enterprise, and Data Center editions, plus **SonarQube Cloud (SonarCloud)**, are commercial and unlock deeper security (taint) analysis and more languages (Freemium / Commercial).

## Tags

`sast` `static-analysis` `code-quality` `quality-gate` `security-hotspots` `sonarqube` `ci-cd`

---

**Contributed by:** @ruimachado23
**Last Updated:** 2026-06-10
