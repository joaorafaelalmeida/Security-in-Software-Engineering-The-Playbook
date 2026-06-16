# Open-Source License Compliance for Engineers

> Audit, attribute, and contain the legal obligations your dependencies impose, before they spread into your product.

## The Problem

When you ship software, your product silently absorbs the legal obligations of every dependency you pull in, including the transitive ones you never chose directly. This is an engineering problem, not just a lawyer problem: the architectural decisions you make (how you link a library, where you draw a process boundary, what you bundle into a release artifact) are exactly what determines which obligations attach and how far they spread.

What can go wrong if you ignore this:

- A single accidental GPL or AGPL dependency in a proprietary codebase can force you to open-source code you intended to keep closed.
- Missing attribution is a breach of the license grant itself, which can terminate your right to use the software at all.
- An AGPL component in a networked service can pull source-disclosure obligations across your whole backend.

In practice, **most real-world violations are missing attribution, not missing source.** Knowing a license exists is not the same as discharging the obligation it carries.

## The Solution

### Overview

License compliance is a repeatable engineering loop: inventory the licenses that enter your product, identify the most restrictive one, discharge the obligations it carries (usually attribution), and use your architecture to contain anything strongly copyleft. Compatibility flows one way, so the most restrictive license present sets the floor for the entire product. The earlier you inventory and the more deliberate your linking and process boundaries, the cheaper this stays.

### Auditing your dependencies

Start with the business question: what licenses enter the product through its dependencies, and which is the most restrictive? You cannot reason about obligations you have not inventoried.

Use the ecosystem's tooling to list licenses:

- **Python:** `pip-licenses`
  ```bash
  pip-licenses --summary
  pip-licenses --format=plain | grep -iE "GPL|LGPL|MPL|AGPL"
  ```
- **Java/Maven:** `license-maven-plugin` (aggregates declared licenses across the dependency tree).

The first command gives you the distribution of licenses; the second flags the copyleft ones that need attention.

Licenses sit on a rough one-way spectrum from permissive to strongly copyleft:

```
MIT / BSD / Apache  ->  MPL  ->  LGPL  ->  GPL  ->  AGPL
(permissive)       (file)   (library)        (network copyleft)
```

- **MIT, BSD, Apache:** permissive. Keep the copyright notice and license text, do almost anything else.
- **MPL:** file-level copyleft. Changes to MPL-covered files must stay open, but the rest of your code is untouched.
- **LGPL:** library-level copyleft. Manageable if the library stays separable (see linking, below).
- **GPL:** strong copyleft. A combined work that includes GPL code becomes GPL.
- **AGPL:** GPL plus a network clause (see the SaaS trap, below).

The critical rule: **compatibility flows one way.** A combined work takes on the most restrictive license it touches. If you mix MIT, Apache, and one LGPL-2.1 component, the combined product must satisfy at least LGPL-2.1 terms. You cannot dilute a strong license by surrounding it with permissive ones. The most restrictive license present sets the floor for the whole product, which is why a single accidental GPL or AGPL dependency in a proprietary codebase is the line you do not want to cross by mistake.

### Attribution

Identifying a license only tells you the obligation. You still have to **discharge it**, and for permissive licenses the obligation is almost always attribution: you must reproduce the copyright notice and license text **in the distribution you hand to the customer**.

"The user can go find it on GitHub" does not satisfy this. The clause puts the duty on the distributor to include the notice, not on the user to hunt for it. A missing notice is a breach of the license grant itself, which can terminate your right to use the software at all.

Generate a consolidated notices file as a build artifact:

```bash
# Python example
pip-licenses --with-license-file --no-license-path --format=markdown \
  --output-file THIRD-PARTY-NOTICES.md
```

Then verify no dependency is missing its license text (the classic compliance gap):

```bash
pip-licenses --format=json --with-license-file --no-license-path | python3 -c "
import json, sys
d = json.load(sys.stdin)
missing = [p['Name'] for p in d if p.get('LicenseText','UNKNOWN') in ('UNKNOWN','')]
print('total:', len(d), '| missing license text:', len(missing))"
```

Modern packages increasingly bundle their license file, so this check often comes back clean, but it is cheap and catches the historical offenders that ship without one.

**Where the notices ship:** wherever the customer can reach them without the source.

- An **"About / Open-Source Licenses" screen** in a GUI or mobile app (this is why every phone app has that buried menu).
- A **`/legal` or `/licenses` route** in a web app.
- A `LICENSES` / `NOTICES` file inside the **installer bundle or container image**.
- `--licenses` output for a CLI.

The notices must travel with the artifact the customer actually receives.

### Derivative works and linking

How you link a library determines whether it becomes part of a single derivative work or stays a separate component, and that distinction drives the obligations.

- **Static linking** physically copies the library's code into your binary. The result is one artifact with no boundary left, so the combined binary is a **single derivative work** and inherits the library's license obligations wholesale.
- **Dynamic linking** keeps the library as a separate, replaceable file loaded at runtime. The library sits on the other side of a runtime boundary and remains swappable.

You can see this directly. Compiling a trivial program both ways:

```bash
gcc usemath.c -o dyn  -lm           # dynamic
gcc usemath.c -o stat -static -lm   # static
```

The dynamic binary is tiny (the library is an external file that shows up in `ldd`); the static binary is far larger (the library has been merged in, and `ldd` reports "not a dynamic executable"). The static build is the single derivative work.

**Why "dynamic linking + LGPL" is the conservative safe path.** The LGPL's central condition is that the end user must be able to **replace the LGPL library with their own version** and still run the program. Dynamic linking satisfies this almost for free, because the library is a swappable file the user can drop a modified build over. Static linking breaks it: the library is fused into your binary, so honoring the "must be replaceable" clause would mean shipping object files or otherwise enabling relinking, a much heavier obligation. When you are unsure whether a copyleft library will infect your proprietary code, keep it behind a dynamic-linking boundary.

### The license firewall

When you must use a strong copyleft component (for example AGPL) in an otherwise proprietary system, you contain its obligations by isolating it behind a **process and network boundary**.

Consider three services where A and C are proprietary and B uses an AGPL component:

```
[ Service A ]      [ Service B ]      [ Service C ]
 proprietary        AGPL               proprietary
        \________ network boundary ________/
   AGPL obligations contained in: Service B only
```

Run Service B as its own process, ideally its own deployable, and let A and C talk to it only over a network protocol (REST, gRPC, a message queue) across an arm's-length API. AGPL's reach stops at that boundary: A and C merely *call* an independent network service, which does not make them derivative works of B. The source-disclosure obligation then applies to Service B alone.

A repository boundary on its own is not enough. Separate repos that get compiled into one running process still form a combined work.

## Common Pitfalls

1. **The AGPL-in-SaaS trap.** A normal GPL obligation triggers only when you **distribute a binary**, so SaaS teams reason (correctly, for plain GPL) that since they never ship anything to the customer, the source-disclosure obligation never fires. AGPL's network clause was written to close exactly this loophole: it treats **serving the software over a network as a form of distribution.** If an AGPL component is part of your service, you owe the complete corresponding source of the whole service to every user who interacts with it over the network, which can force you to open-source your proprietary backend. AGPL dependencies are commonly banned outright in commercial SaaS dependency policies, and your audit should specifically flag them.
2. **Missing attribution.** Identifying a license is not discharging it. Shipping a product without the bundled copyright notices and license texts is the most common violation in practice; generate `THIRD-PARTY-NOTICES` as a build artifact and confirm it travels with the artifact the customer receives.
3. **Static-linking a copyleft library.** Static linking fuses the library into your binary as a single derivative work, inheriting its obligations wholesale and breaking LGPL's "user must be able to replace the library" condition. Prefer dynamic linking when a copyleft library is involved.
4. **Firewall failure via in-process coupling.** The license firewall only holds when the boundary is genuinely arm's-length. It fails when you turn B into an **in-process plugin or shared library** that A loads into its own address space (now A and B are one running program and A is arguably a derivative work, AGPL and all), or when a tightly-coupled **sidecar shares memory** with A (a shared-memory IPC segment, or an `LD_PRELOAD`ed `.so`) instead of using a clean network call. Shared address space means there is no real boundary, and the copyleft leaks straight into the proprietary services.

## When to Apply

- **Always:** Before distributing or deploying any product that pulls in third-party dependencies. Inventory licenses and ship attribution with every release artifact.
- **Recommended:** When introducing a new dependency, especially a copyleft one, decide its linking model and process boundary before you integrate it.
- **Consider:** When a strongly copyleft component (GPL, AGPL) is genuinely the best technical fit for a proprietary system, design a license firewall around it rather than rejecting it outright.

## Standards & Compliance

- **OWASP Top 10:** A06:2021 Vulnerable and Outdated Components. License compliance lives alongside vulnerability management in the supply-chain category, and the same dependency inventory serves both.
- **SPDX (Software Package Data Exchange):** the standard identifiers (`MIT`, `Apache-2.0`, `LGPL-2.1-only`, `AGPL-3.0-only`, etc.) and a license-data exchange format. Use SPDX identifiers consistently in your audits and notices.
- **SBOM (Software Bill of Materials), CycloneDX and SPDX formats:** a machine-readable inventory of every component and its license. Generating an SBOM as a build artifact makes both license auditing and the attribution file repeatable and automatable rather than a one-off manual chore.

## Further Reading

- [SPDX License List](https://spdx.org/licenses/)
- [OWASP Top 10: A06:2021 Vulnerable and Outdated Components](https://owasp.org/Top10/A06_2021-Vulnerable_and_Outdated_Components/)
- [CycloneDX SBOM Standard](https://cyclonedx.org/)
- [GNU Licenses and FAQ](https://www.gnu.org/licenses/gpl-faq.html)
- [pip-licenses Documentation](https://pypi.org/project/pip-licenses/)

## Tags

`license-compliance` `sbom` `supply-chain` `copyleft` `agpl` `open-source`

---

**Contributed by:** Henrique Teixeira
**Last Updated:** 2026-06-16
**Difficulty Level:** Intermediate
**Impact:** Medium
