# SAST as a Shift-Left Control: Pre-commit Hooks and CI Gating

> Catch dangerous code patterns with pattern-based static analysis both locally before a commit and on the server before a merge, where they are cheapest to fix.

## The Problem

Insecure patterns such as dynamic code execution and shell-based command construction are easy to write and expensive to discover late. The longer an injection bug survives, the more it costs to fix: a flaw caught on a developer's machine is a one-line edit, while the same flaw caught in production is an incident.

Relying on a single control fails in predictable ways:

- Local-only scanning is trivially bypassed (`git commit --no-verify`) and never runs for a teammate who never set it up.
- Server-only scanning gives feedback late, after push, when the developer has already moved on.

Without a layered approach, vulnerable code reaches the shared branch and the team pays remediation cost instead of prevention cost.

## The Solution

### Overview

Semgrep is a pattern-based static analysis tool that flags insecure code by matching it against rule packs. Because it does pattern matching and never executes your code, it is fast and safe to run repeatedly. Use it as a shift-left control wired into two layers that cover different failure modes:

- a **local pre-commit hook** for instant feedback before code enters history;
- a **CI gate on pull requests** that, with branch protection, cannot be skipped and applies uniformly to everyone.

Hooks optimize for speed and developer experience; CI optimizes for enforcement. Run both as defense in depth: the hook keeps most issues out of history, and CI is the safety net that guarantees nothing vulnerable merges.

### Implementation

#### Layer 1: Local pre-commit hook

The [`pre-commit`](https://pre-commit.com/) framework runs registered hooks against your staged files before Git finalizes a commit. Add Semgrep as one of those hooks by creating `.pre-commit-config.yaml` at the repo root:

```yaml
repos:
  - repo: https://github.com/semgrep/pre-commit
    rev: 'v1.150.0'
    hooks:
      - id: semgrep
        name: Semgrep SAST
        args: ['--config', 'p/default', '--error', '--skip-unknown-extensions']
```

Then install the framework and wire the hook into your local `.git/hooks/`:

```bash
pip install pre-commit
pre-commit install
```

What the flags do:

- `--config p/default` runs Semgrep's curated default ruleset (a broad set of security and correctness rules).
- `--error` makes findings return a non-zero exit code, so the commit is blocked.
- `--skip-unknown-extensions` avoids scanning file types Semgrep has no rules for.

From now on, every `git commit` scans your staged code first. To scan the whole tree on demand (useful right after setup), run `pre-commit run --all-files`. The first run downloads the Semgrep environment and rules, so it is slower; later runs are fast.

#### Layer 2: CI gate on pull requests

A hook only protects the machine it is installed on. A CI job protects the whole team. Add a GitHub Actions workflow at `.github/workflows/semgrep.yml` that runs on pull requests:

```yaml
name: SAST Pipeline (Semgrep)

on:
  pull_request:
    branches: [ "main" ]

jobs:
  security-scan:
    name: Semgrep Analysis
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: "p/default"
          generateSarif: "1"
```

This checks out the PR code and runs the same `p/default` ruleset. `generateSarif: "1"` emits a SARIF report, which GitHub can surface in the Security tab and as inline annotations.

A failing scan alone only turns the check red; it does not stop a merge by itself. To make the gate blocking, enable branch protection on `main` (Settings, Branches, Add rule) and mark the Semgrep check as a required status check. With that in place, the merge button stays disabled until the scan passes, so vulnerable code physically cannot reach the protected branch.

## Code Examples

These two representative findings from the default ruleset both map to OWASP A03:2021 Injection. The general lesson is the same in each case: avoid passing untrusted input into interpreters (code evaluators, shells, SQL), and prefer APIs that separate code from data.

### Bad Practice (Vulnerable)

```python
# Flagged: python.lang.security.audit.dynamic-execution
discount = eval(user_input)   # user input evaluated as code
```

```javascript
// Flagged: javascript.lang.security.audit.dangerous-system-command
const command = "ping -c 4 " + userIP;
exec(command, (error, stdout, stderr) => { ... });
```

**Why this is problematic:**

- If `user_input` is attacker-controlled, `eval()` evaluates it as code, which is code injection.
- `exec` runs its argument through a shell, so input like `8.8.8.8; rm -rf /` would execute arbitrary commands.

### Good Practice (Secure)

```python
discount = int(user_input)    # safe: parses, does not execute
```

```javascript
execFile('ping', ['-c', '4', userIP], (error, stdout, stderr) => { ... });
```

**Why this works:**

- When you only need a number, `int()` parses the input instead of evaluating it, so no code path is exposed.
- `execFile` takes the command and an argument array, so arguments are passed directly and never parsed by a shell.

## Common Pitfalls

1. **Hooks are bypassable.** `git commit --no-verify` skips every hook. Treat the local hook as a convenience, not a guarantee, and rely on the CI gate plus branch protection for enforcement.
2. **Not every developer has hooks installed.** `pre-commit install` is per-clone and per-machine. New contributors and fresh checkouts start with no hook, so never assume local scanning happened. The server-side gate is what makes coverage uniform.
3. **A red CI check does not block on its own.** Without branch protection marking the job as a required status check, a failing Semgrep run still leaves the merge button enabled. Configure the required check explicitly.
4. **Scan scope and speed.** Pre-commit only scans staged files by default, which is fast but can miss issues in untouched code; periodically run `pre-commit run --all-files`. CI scans more and takes longer, so budget for it in your pipeline.
5. **Pattern matching has limits.** Semgrep is strong at known dangerous patterns (`eval`, `exec`, and similar) but will not catch complex logic or business-rule flaws. Pair it with code review and other testing rather than treating a green scan as proof of security.

## When to Apply

- **Always:** On any shared repository where multiple contributors merge code, the CI gate with branch protection is the authoritative control that prevents vulnerable code from reaching the protected branch.
- **Recommended:** Add the local pre-commit hook for every developer to shorten the feedback loop and keep most issues out of history before they are ever pushed.
- **Consider:** For small or solo projects, even the pre-commit hook alone establishes good habits early, though the CI gate is what scales the practice to a team.

## Standards & Compliance

- **OWASP Top 10:** A03:2021 - Injection. Both example findings (dynamic execution and shell command construction) fall under this category.
- **CWE-95:** Improper Neutralization of Directives in Dynamically Evaluated Code (`eval` injection).
- **CWE-78:** Improper Neutralization of Special Elements used in an OS Command (command injection).

## Further Reading

- [OWASP Top 10 (2021), A03:2021 Injection](https://owasp.org/Top10/A03_2021-Injection/)
- [Semgrep Documentation](https://semgrep.dev/docs/)
- [Semgrep pre-commit integration](https://semgrep.dev/docs/extensions/pre-commit)
- [pre-commit framework](https://pre-commit.com/)

## Tags

`sast` `semgrep` `shift-left` `ci-cd` `pre-commit` `owasp-top-10`

---

**Contributed by:** Henrique Teixeira
**Last Updated:** 2026-06-16
**Difficulty Level:** Intermediate
**Impact:** High
