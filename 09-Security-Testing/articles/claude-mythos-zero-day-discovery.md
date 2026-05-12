# Claude and AI-Assisted Zero-Day Discovery: Project Mythos

> How Anthropic's Claude has been used to autonomously discover and exploit previously-unknown vulnerabilities in widely-deployed open-source software.

## Metadata

- **Author(s):** Anthropic Red Team (primary sources); secondary coverage by The Hacker News
- **Publication:** red.anthropic.com / thehackernews.com
- **Date:** 2026-02-05 (initial post) and 2026-04-07 (Mythos preview)
- **Reading Time:** ~20 minutes (across the cited posts)
- **Level:** Intermediate
- **Link:** [0-Days (Feb 2026)](https://red.anthropic.com/2026/zero-days/) · [Claude Mythos Preview (Apr 2026)](https://red.anthropic.com/2026/mythos-preview/)

## Summary

Through 2025-2026 Anthropic published a series of posts documenting that Claude can find real, exploitable vulnerabilities in mature open-source projects, not as a code-review assistant pointing at suspicious lines, but as an autonomous agent that triages a codebase, hypothesises a bug class, builds a proof-of-concept, and verifies exploitability.

The February 2026 "0-Days" post claimed 500+ validated high-severity findings across audited OSS, with publicly named patches landing in **GhostScript**, **OpenSC**, and **CGIF**. The April 2026 "Claude Mythos Preview" announced **Project Glasswing** and disclosed three headline cases: a 17-year-old FreeBSD NFS RCE in `RPCSEC_GSS` (**CVE-2026-4747**) that Claude discovered *and* exploited end-to-end, a 27-year-old OpenBSD TCP SACK denial-of-service, and a 16-year-old FFmpeg H.264 heap out-of-bounds write patched in FFmpeg 8.1.

The pattern matters because these are not lint-style findings, they are deep, multi-step memory-safety and protocol bugs that survived decades of human review. For AppSec teams the implication is twofold: defensive use (running these agents against your own code before adversaries do) is now a credible part of the SDLC, and the threat model has to assume well-resourced attackers are doing the same.

## Key Takeaways

1. **Autonomous discovery is no longer hypothetical.** The Glasswing bugs were found and weaponised without human-in-the-loop hint-giving. Treat "an AI found a 0-day in our code" as a realistic 2026 scenario, not a 2030 one.
2. **Old code is the soft target.** Every named bug was 16+ years old. LLM agents read at constant cost regardless of how long a file has sat untouched, so legacy modules that humans avoid are exactly where AI excels.
3. **Protocol-level and memory-safety bugs dominate the publicly-disclosed set.** RPC parsers, kernel network stacks, multimedia codecs: the classes that need full-program reasoning, not the ones a regex would catch.
4. **The capability is asymmetric until you adopt it.** Defenders who do not run AI-assisted review on their own codebase concede the time-to-discovery race to anyone who does.

## Why This Matters

For AppSec/DevSecOps practice this collapses two assumptions that most secure-SDLC programs were built on:

- **"Mature code is implicitly safer."** It still is *relative to new code*, but the gap is shrinking fast: a 17-year-old FreeBSD bug surviving every prior audit until an LLM agent walked the call graph end-to-end is exactly the kind of finding that used to take a dedicated researcher months.
- **"Static analysis catches the easy stuff and humans catch the rest."** Claude-class agents now sit between those tiers: they reason across files, model attacker capabilities, and produce working PoCs. They belong in the testing pipeline next to fuzzers and SAST, not as a replacement for either.

## Practical Applications

- **Pre-release agentic review:** Run Claude (via the API or Claude Code's `/security-review` skill) against changed subsystems before a release branch is cut. Treat its output like a fuzzer corpus: triage first, don't merge blindly.
- **Legacy module sweeps:** Pick the 3-5 oldest, least-touched modules in your repo and have an agent specifically hunt for memory-safety / parser / auth-boundary bugs there. That's where the asymmetric value is.
- **PoC-first triage:** Require any AI-surfaced finding to come with a reproducer before it consumes engineering time. The Glasswing posts emphasise verified exploitability for this reason.

## Notable Quotes

> "Each of these issues survived decades of human review. The agent did not." (paraphrased from the Mythos Preview post on the FreeBSD/OpenBSD/FFmpeg trio.)

## Related Topics

- Autonomous offensive-security agents and dual-use risk
- Coordinated disclosure for AI-discovered vulnerabilities
- Fuzzing and symbolic execution as complementary techniques
- LLM-assisted code review in CI

## Further Reading

- [Anthropic Red: 0-Days (Feb 2026)](https://red.anthropic.com/2026/zero-days/)
- [Anthropic Red: Claude Mythos Preview (Apr 2026)](https://red.anthropic.com/2026/mythos-preview/)
- [The Hacker News: Anthropic's Claude Mythos finds… (Apr 2026)](https://thehackernews.com/2026/04/anthropics-claude-mythos-finds.html)
- [Project Zero: From Naptime to Big Sleep (Nov 2024)](https://projectzero.google/2024/10/from-naptime-to-big-sleep.html). Google's parallel work using Gemini to find a SQLite memory-safety bug.
- [The Hacker News: Big Sleep stops in-the-wild exploitation (Jul 2025)](https://thehackernews.com/2025/07/google-ai-big-sleep-stops-exploitation.html). First AI-foiled live exploit (CVE-2025-6965).

## Critical Analysis

**Strengths.** The Anthropic posts include named CVEs, named projects, and patched releases. That is a substantively different bar from earlier "AI found bugs!" claims that were either toy CTF challenges or undisclosed. The parallel work from Google Project Zero corroborates the underlying capability, so the Anthropic claims are not a single-vendor outlier.

**Limitations.** The "500+" and "thousands" figures are vendor-reported and most of the underlying findings are under embargo, so independent verification is currently limited to the named CVEs. The posts also do not disclose compute cost or false-positive rate, both of which materially affect whether this is reproducible by smaller teams or only by Anthropic-scale infrastructure. Treat the headline numbers as a directional signal, not a benchmark.

**Aging.** The capability is moving fast. By the time you read this entry the named projects, model versions, and CVE IDs may have been superseded.

## Tags

`ai-security` `vulnerability-discovery` `zero-day` `llm-agents` `claude` `security-testing` `red-team`

---

**Contributed by:** Ricardo Dias
**Last Updated:** 2026-05-12
**Relevance:** High. AI-assisted vulnerability discovery is a 2026-current topic with named CVEs and patched releases; directly informs how AppSec teams should adapt their testing pipeline and supply-chain threat model.
