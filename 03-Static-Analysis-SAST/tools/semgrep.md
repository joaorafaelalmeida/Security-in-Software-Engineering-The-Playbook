# Semgrep

> A fast, open-source static analysis engine that finds bugs and security issues using rules that look like the code they match.

## Overview

Semgrep ("semantic grep") is a lightweight, polyglot SAST tool that scans source code for security and correctness issues. Its defining feature is the rule syntax: instead of writing complex AST queries, you write a pattern that **looks like the code you want to find**, with metavariables (`$X`) for the parts that vary. This makes it unusually approachable — teams can write custom rules in minutes — while still supporting cross-function taint (data-flow) analysis in the Pro engine.

It supports 30+ languages, runs in seconds to minutes, requires no build step for most languages, and emits SARIF for clean CI integration. It has become the de-facto default for the "fast pattern SAST" stage of a pipeline.

## Key Features

- **Pattern-as-code rules:** Rules resemble the target code, lowering the barrier to writing and auditing them.
- **Polyglot:** 30+ languages including Python, JavaScript/TypeScript, Java, Go, Ruby, C#, PHP, and more.
- **Curated rule registry:** Thousands of community and vendor rules via `--config auto` or named packs (`p/ci`, `p/owasp-top-ten`, `p/python`).
- **Taint/data-flow mode:** Source-to-sink tracking to catch injection across functions (deeper in the Pro engine).
- **Diff-aware scanning:** `semgrep ci` scans only what a PR changes, enabling baseline-style gating.
- **SARIF output:** First-class integration with GitHub/GitLab code scanning and triage tools.

## Use Cases

- Pre-commit and PR-stage SAST where speed and low false positives matter most.
- Enforcing organization-specific secure-coding rules ("never call this internal API without auth").
- Catching OWASP Top 10 patterns (injection, XSS, SSRF) across a polyglot monorepo.
- Banning dangerous functions or risky patterns as a team policy gate.

## Installation & Setup

### Prerequisites
- Python 3.8+ (for the pip install) or Docker.
- No build/compile step required for most languages.

### Installation

```bash
# pip
python3 -m pip install semgrep

# Homebrew
brew install semgrep

# Docker
docker run --rm -v "${PWD}:/src" semgrep/semgrep semgrep --config auto
```

## Usage Examples

### Basic Usage

```bash
# Scan the current directory with auto-selected rules
semgrep --config auto .

# Use a specific curated ruleset
semgrep --config p/owasp-top-ten --config p/secrets .
```

### Advanced Usage

```bash
# CI mode: diff-aware, fails the build on findings, emits SARIF
semgrep ci --config auto --sarif --output semgrep.sarif

# Run a single custom rule file with verbose output
semgrep --config ./rules/no-eval.yml --verbose ./src
```

### Configuration Example

```yaml
# rules/no-eval.yml — a custom rule that looks like the code it catches
rules:
  - id: dangerous-eval
    languages: [python]
    severity: ERROR
    message: Avoid eval() on dynamic input; it enables code injection (CWE-95).
    metadata:
      cwe: "CWE-95: Improper Control of Generation of Code"
      owasp: "A03:2021 - Injection"
    pattern: eval(...)
```

## Pros and Cons

### Advantages ✅
- Extremely fast; no build step for most languages.
- Easiest custom-rule authoring of any mainstream SAST tool.
- Strong free/OSS tier with a large curated rule registry.
- Excellent CI ergonomics (diff-aware, SARIF, exit codes).

### Limitations ⚠️
- Deep cross-file taint analysis is strongest in the paid **Pro** engine.
- Pattern rules can miss flows that span complex indirection.
- Very large monorepos may need rule scoping to stay fast.

## Integration

```yaml
# GitHub Actions — PR gate
- name: Semgrep
  run: semgrep ci --config auto --sarif --output semgrep.sarif
- uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: semgrep.sarif
```

Also integrates via a pre-commit hook (`https://github.com/semgrep/pre-commit`), GitLab CI templates, and IDE extensions.

## Alternatives

- **CodeQL** — deeper semantic/data-flow analysis; steeper learning curve and needs build context.
- **SonarQube** — broader code-quality + security platform with dashboards and historical trends.
- **Bandit** — Python-only, simpler, good complement in pre-commit.

## Resources

- [Official Documentation](https://semgrep.dev/docs/)
- [Rule Registry](https://semgrep.dev/explore)
- [GitHub Repository](https://github.com/semgrep/semgrep)
- [Interactive Rule Playground](https://semgrep.dev/playground)

## License

Open Source (LGPL 2.1) core engine and rules; commercial **Semgrep Pro/AppSec Platform** adds deep cross-file taint analysis, managed scanning, and triage features (Freemium).

## Tags

`sast` `static-analysis` `security-scanner` `pattern-matching` `polyglot` `ci-cd`

---

**Contributed by:** @ruimachado23
**Last Updated:** 2026-06-10
