# OWASP Web Security Testing Guide v4.2

> Official OWASP guide for structured web application security testing.

## Metadata

- **Author(s):** OWASP Web Security Testing Guide Project
- **Publication:** OWASP Foundation
- **Date:** 2020-12-03
- **Level:** Beginner to Intermediate
- **Local PDF:** [owasp-web-security-testing-guide-v4.2.pdf](./owasp-web-security-testing-guide-v4.2.pdf)
- **Official Project Page:** [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- **Official PDF Source:** [Download v4.2 PDF](https://github.com/OWASP/wstg/releases/download/v4.2/wstg-v4.2.pdf)
- **License:** Creative Commons Attribution-ShareAlike 4.0

## Summary

The OWASP Web Security Testing Guide (WSTG) is a structured guide for testing web applications and web services. It is useful for learning DAST because it explains what should be tested, why it matters, and how different web security test categories fit together.

While OWASP ZAP can automate parts of dynamic testing, the WSTG gives the broader methodology. It helps testers avoid treating a scanner report as the full security assessment. Automated tools are useful, but web security testing also needs planning, manual validation, and understanding of application behaviour.

Version 4.2 is available as a versioned PDF, which makes it practical to cite and keep in a repository. Versioned references are useful because testing scenario identifiers and content can change between releases.

## Key Takeaways

1. **DAST needs a testing method**
   - Running a scanner is not enough. Testers need a structured view of authentication, authorization, input validation, session management, error handling, cryptography, and business logic.

2. **Automated and manual testing complement each other**
   - Tools like OWASP ZAP help discover runtime issues, but manual review is still needed for workflows, access control, and business logic.

3. **Evidence matters**
   - Good testing should record the request, response, payload, affected endpoint, risk, and suggested remediation.

4. **Testing should be repeatable**
   - The WSTG gives categories and identifiers that can be reused in reports, checklists, and CI/CD testing strategies.

## Why This Matters

This PDF is useful for an introductory Software Engineering Security course because it connects practical testing with software engineering process.

It helps answer questions like:

- What should be tested in a web application?
- Which tests can be automated with DAST?
- Which areas still need manual validation?
- How should findings be documented?
- How do we move from a scanner alert to a real security finding?

## Practical Applications

- Use it as a checklist when planning a DAST assessment.
- Map ZAP findings to WSTG testing categories.
- Use WSTG identifiers when writing security reports.
- Use the guide to decide which tests belong in CI/CD and which belong in manual testing.
- Compare scanner output with manual testing coverage.

## Relation to OWASP ZAP

OWASP ZAP can help with several WSTG areas, especially:

- information gathering;
- configuration and deployment checks;
- input validation testing;
- error handling checks;
- session and authentication testing when configured correctly;
- fuzzing and active scanning.

However, ZAP does not replace the WSTG. The WSTG explains the testing strategy; ZAP is one tool that can help execute parts of it.

## Related Topics

- Dynamic Application Security Testing
- OWASP ZAP
- Manual Security Testing
- Web Application Testing
- Penetration Testing
- Security Reporting

## Further Reading

- [OWASP ZAP Automation Framework](https://www.zaproxy.org/docs/automate/automation-framework/)
- [OWASP ZAP Docker Documentation](https://www.zaproxy.org/docs/docker/)
- [OWASP Web Security Testing Guide Project](https://owasp.org/www-project-web-security-testing-guide/)
- [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/)

## Tags

`dast` `owasp` `wstg` `web-security-testing` `penetration-testing` `security-testing`

---

**Contributed by:** João Varela  
**Last Updated:** 2026-05-03  
**Relevance:** High - this is one of the main OWASP references for structured web application security testing.
