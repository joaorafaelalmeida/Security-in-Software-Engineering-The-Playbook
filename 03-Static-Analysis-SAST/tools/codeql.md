# CodeQL

> GitHub's semantic code analysis engine that lets you query code like a database to find security vulnerabilities through deep data-flow analysis.

## Overview

CodeQL treats code as **data**. It builds a relational database from a codebase, then runs queries written in the declarative QL language against that database to find patterns — most importantly, **taint-tracking** paths from untrusted sources to dangerous sinks. This makes it one of the most powerful engines for finding real, exploitable data-flow vulnerabilities (injection, SSRF, unsafe deserialization) that span multiple functions and files.

It powers GitHub **code scanning** and is free for public repositories and for research/open-source use. Because it performs deep semantic analysis, it typically needs to observe a build (for compiled languages) and runs slower than pattern scanners — placing it firmly in the "scheduled / deep CI" stage of a pipeline rather than pre-commit.

## Key Features

- **Taint tracking / data-flow analysis:** Follows untrusted input from source to sink across the whole program.
- **Query-as-data model:** Code is compiled into a database queried with the QL language; results are precise and explainable (full path shown).
- **Curated query suites:** `security-extended` and `security-and-quality` packs maintained by GitHub Security Lab.
- **Multi-language:** C/C++, C#, Go, Java/Kotlin, JavaScript/TypeScript, Python, Ruby, Swift.
- **Native GitHub code scanning:** Findings render inline in PRs with the full data-flow path.
- **Custom queries:** Write organization-specific queries to model your own sources/sinks.

## Use Cases

- Deep, scheduled scans for high-impact injection and data-flow vulnerabilities.
- Free, high-quality SAST for open-source projects via GitHub code scanning.
- Variant analysis: after one bug is found, query for every other instance of the same pattern across the codebase.
- Security research and writing precise custom detections.

## Installation & Setup

### Prerequisites
- A GitHub repository (for the Actions-based workflow) **or** the CodeQL CLI for local/other-CI use.
- For compiled languages: the ability to build the project so CodeQL can observe compilation.

### Installation

```bash
# Easiest: enable via GitHub Actions (no local install needed).
# For local/CLI use, download the CodeQL CLI bundle:
gh extensions install github/gh-codeql   # GitHub CLI extension
gh codeql --version
```

## Usage Examples

### Basic Usage (GitHub Actions)

```yaml
# .github/workflows/codeql.yml
name: codeql
on:
  schedule:
    - cron: "0 2 * * *"
jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: python, javascript-typescript
          queries: security-extended
      - uses: github/codeql-action/analyze@v3
```

### Advanced Usage (CLI)

```bash
# Build a database, then run the security suite, emitting SARIF
codeql database create db --language=python --source-root=.
codeql database analyze db codeql/python-queries:codeql-suites/python-security-extended.qls \
  --format=sarif-latest --output=codeql.sarif
```

### Custom Query Example

```ql
/** Find Python eval() calls reachable from untrusted input. */
import python
import semmle.python.dataflow.new.TaintTracking
import semmle.python.ApiGraphs

from DataFlow::Node sink
where sink = API::builtin("eval").getACall().getArg(0)
select sink, "Untrusted data may reach eval() — potential code injection."
```

## Pros and Cons

### Advantages ✅
- Best-in-class data-flow/taint analysis with explainable result paths.
- Free for public repos and OSS; deeply integrated with GitHub.
- Powerful variant analysis to eliminate whole bug classes.
- Custom QL lets you model your own security properties precisely.

### Limitations ⚠️
- Steep learning curve for writing custom QL.
- Needs build observation for compiled languages; slower than pattern scanners.
- Heavier setup and compute — not suited to pre-commit/every-push.
- Commercial use on private repos requires GitHub Advanced Security licensing.

## Integration

```yaml
# Combine with a fast pattern scanner: Semgrep in PRs, CodeQL on a schedule.
on:
  pull_request:        # -> run Semgrep here (fast gate)
  schedule:            # -> run CodeQL here (deep analysis)
    - cron: "0 2 * * *"
```

Findings appear in the GitHub Security tab; SARIF can be exported to other dashboards.

## Alternatives

- **Semgrep (Pro)** — easier rules and faster; Pro engine adds cross-file taint.
- **Checkmarx / Fortify** — enterprise commercial SAST with broad language and compliance coverage.
- **Snyk Code** — fast, ML-assisted SAST with strong IDE/PR integration.

## Resources

- [CodeQL Documentation](https://codeql.github.com/docs/)
- [CodeQL Query Help](https://codeql.github.com/codeql-query-help/)
- [GitHub Security Lab](https://securitylab.github.com/)
- [CodeQL on GitHub](https://github.com/github/codeql)

## License

Free for analysis of open-source code and academic research; commercial/private-repo use requires **GitHub Advanced Security** (Commercial). The CodeQL CLI and queries are source-available under the GitHub CodeQL terms.

## Tags

`sast` `static-analysis` `taint-analysis` `data-flow` `codeql` `github` `code-scanning`

---

**Contributed by:** @ruimachado23
**Last Updated:** 2026-06-10
