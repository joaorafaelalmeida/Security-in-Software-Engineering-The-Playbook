# AFL++ (American Fuzzy Lop++)

> A state-of-the-art coverage-guided fuzzer that finds crashes, memory corruption, and logic bugs by automatically mutating inputs and tracking which code paths each mutation exercises.

## Overview

AFL++ is the community-maintained successor to Google's AFL. It instruments the target binary at compile time so that every test input reports which edges in the control-flow graph it exercised. Inputs that reach new edges are kept; inputs that do not are discarded. This feedback loop drives the fuzzer toward unexplored corners of the code far faster than random mutation alone.

AFL++ is used extensively in security research, OSS-Fuzz, and vulnerability disclosure programs. It has contributed to the discovery of thousands of CVEs across browsers, media libraries, network parsers, and language runtimes.

## Key Features

- **Coverage-guided mutation**: edge-coverage feedback ensures the fuzzer explores new code paths over time rather than hammering already-covered paths
- **LLVM instrumentation mode**: `afl-cc` / `afl-clang-fast` provide faster, more precise instrumentation than classic assembly-level patching
- **Persistent mode**: eliminates fork overhead by resetting state inside the process, yielding 10–100× higher throughput for library targets
- **Sanitizer integration**: pairs natively with AddressSanitizer, UndefinedBehaviorSanitizer, and MemorySanitizer to catch bugs that would not produce a visible crash on their own
- **Corpus minimization and deduplication**: `afl-cmin` and `afl-tmin` shrink large seed sets and reduce findings to minimal reproducers
- **Parallel fuzzing**: `afl-whatsup` and the `-M`/`-S` flags coordinate multiple fuzzer instances across cores and machines
- **Custom mutators**: Python or C API for domain-specific mutation strategies (e.g., structured protocol fuzzing)
- **QEMU and Unicorn modes**: instrument closed-source or firmware binaries without recompilation

## Use Cases

- Finding memory-safety bugs (buffer overflows, use-after-free, heap corruption) in C and C++ code
- Fuzzing parsers, decoders, and format handlers (image, audio, archive, network protocol)
- Regression fuzzing - catching bugs reintroduced by refactors before they reach production
- Continuous fuzzing in CI on a scheduled basis alongside SAST and DAST

## Installation & Setup

### Prerequisites

- Linux (recommended); macOS supported with limitations
- `clang` and `llvm` (version 11 or later recommended)
- `make`, `git`

### Installation from source

```bash
git clone https://github.com/AFLplusplus/AFLplusplus
cd AFLplusplus
make distrib
sudo make install
```

### Installation via package manager (Debian/Ubuntu)

```bash
sudo apt-get install afl++
```

Verify the install:

```bash
afl-fuzz --version
```

## Usage Examples

### Basic Usage

Compile the target with AFL++ instrumentation:

```bash
CC=afl-clang-fast CXX=afl-clang-fast++ ./configure
make
```

Or for a single file:

```bash
afl-clang-fast -o target target.c
```

Run the fuzzer:

```bash
mkdir -p corpus findings
echo "valid_seed_input" > corpus/seed1
afl-fuzz -i corpus/ -o findings/ -- ./target @@
```

`@@` is replaced with the path to each generated test file. Results land in `findings/crashes/` and `findings/hangs/`.

### With AddressSanitizer

```bash
AFL_USE_ASAN=1 afl-clang-fast -o target target.c
afl-fuzz -i corpus/ -o findings/ -- ./target @@
```

ASan catches heap overflows, stack overflows, and use-after-free bugs that would otherwise produce silent memory corruption.

### Persistent Mode (higher throughput)

For library targets, wrap the entry point in a persistent loop instead of forking on each input:

```c
#include "afl-fuzz.h"

int main(void) {
    __AFL_INIT();
    while (__AFL_LOOP(10000)) {
        uint8_t buf[1024];
        ssize_t len = read(0, buf, sizeof(buf));
        if (len > 0) {
            target_parse(buf, (size_t)len);
        }
    }
    return 0;
}
```

### Parallel Fuzzing

```bash
# Main instance
afl-fuzz -i corpus/ -o findings/ -M main -- ./target @@

# Secondary instances (run in separate terminals or jobs)
afl-fuzz -i corpus/ -o findings/ -S worker1 -- ./target @@
afl-fuzz -i corpus/ -o findings/ -S worker2 -- ./target @@

# Check status across all instances
afl-whatsup findings/
```

### Corpus Minimization

Before a long run, deduplicate and minimize the seed corpus:

```bash
afl-cmin -i raw_corpus/ -o minimized_corpus/ -- ./target @@
```

Minimize a single crashing input to its smallest reproducer:

```bash
afl-tmin -i findings/crashes/id:000001 -o minimal_crash -- ./target @@
```

### Configuration Example

```bash
# Recommended environment for a CI fuzz job
export AFL_SKIP_CPUFREQ=1        # suppress CPU governor warning in containers
export AFL_NO_UI=1               # disable interactive TUI for log-friendly output
export AFL_AUTORESUME=1          # resume a prior run if findings/ already exists

afl-fuzz -i corpus/ -o findings/ -t 5000 -m 512 -- ./target @@
```

## Pros and Cons

### Advantages

- Mature ecosystem with extensive documentation, academic backing, and active maintenance
- Coverage feedback makes it significantly more effective than purely random fuzzing
- Persistent mode and parallel fuzzing scale well to multi-core CI environments
- Sanitizer integration finds latent bugs that crash-only detection would miss
- QEMU mode extends reach to binaries without source

### Limitations

- Primarily targets C and C++ code; fuzzing managed-language runtimes requires extra harness work
- Instrumentation requires recompilation; closed-source targets need QEMU mode with reduced throughput
- Does not understand high-level application semantics. Fuzzing structured protocols or authenticated APIs needs custom mutators or companion tooling
- Finding a crash is only the start; triage and root-cause analysis require additional effort

## Integration

### GitHub Actions (scheduled nightly fuzz job)

```yaml
name: nightly-fuzz

on:
  schedule:
    - cron: "0 1 * * *"

jobs:
  fuzz:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install AFL++
        run: sudo apt-get install -y afl++

      - name: Build instrumented target
        run: |
          AFL_USE_ASAN=1 afl-clang-fast -o fuzz_target fuzz/fuzz_target.c src/parser.c

      - name: Run fuzzer (10 minutes)
        run: |
          mkdir -p corpus findings
          AFL_NO_UI=1 timeout 600 afl-fuzz -i corpus/ -o findings/ -- ./fuzz_target @@ || true

      - name: Check for crashes
        run: |
          if [ -n "$(ls findings/crashes/ 2>/dev/null | grep -v README)" ]; then
            echo "Crashes found — uploading artifacts"
            exit 1
          fi

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: afl-crashes
          path: findings/crashes/
```

## Alternatives

- **libFuzzer** - LLVM in-process fuzzer; tighter sanitizer integration, no separate process; often used together with AFL++ via AFL++'s `libfuzzer` mode
- **Honggfuzz** - multi-process, hardware-feedback (Intel PT) support; strong for persistent mode network targets
- **Atheris** - libFuzzer-based Python fuzzer from Google; the right choice when the target code is Python
- **OSS-Fuzz** - Google's continuous fuzzing infrastructure; builds on AFL++ and libFuzzer; free for open-source projects

## Resources

- [Official Documentation](https://aflplus.plus/)
- [GitHub Repository](https://github.com/AFLplusplus/AFLplusplus)
- [AFL++ Technical Whitepaper](https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/afl-fuzz_approach.md)
- [OSS-Fuzz Integration Guide](https://google.github.io/oss-fuzz/)
- [Fuzzing Book (free online)](https://www.fuzzingbook.org/)

## License

Apache License 2.0

## Tags

`fuzzing` `coverage-guided` `afl` `memory-safety` `security-testing` `c` `cpp`

---

**Contributed by:** @odraude23
**Last Updated:** 2026-06-17
