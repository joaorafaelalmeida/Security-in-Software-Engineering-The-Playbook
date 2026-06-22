# libFuzzer

> LLVM's in-process, coverage-guided fuzzer (fast, sanitizer-native, and the foundation of OSS-Fuzz and cargo-fuzz).

## Overview

libFuzzer is a fuzzing engine built into the LLVM/Clang toolchain. Unlike AFL++, which forks a new process for each test input, libFuzzer runs entirely inside the target process (the fuzzer loop calls the harness function directly in a tight cycle). This eliminates process-creation overhead and allows libFuzzer to execute hundreds of thousands of iterations per second on fast targets.

libFuzzer instruments the target at compile time with `-fsanitize=fuzzer`, which injects coverage instrumentation and links the fuzzing engine into the final binary. The resulting binary is both the fuzzer and the target in one executable, running it without arguments starts the fuzzing loop. Passing a file path replays that input.

libFuzzer is the engine behind Google's **OSS-Fuzz** continuous fuzzing infrastructure and the Rust **cargo-fuzz** toolchain. AFL++ can also run libFuzzer-compatible harnesses natively, making harnesses written to the `LLVMFuzzerTestOneInput` interface portable across both tools.

## Key Features

- **In-process execution**: no fork overhead. The fuzzing loop calls the harness function directly, enabling very high iteration throughput on fast targets
- **Sanitizer-native**: designed to run with AddressSanitizer, MemorySanitizer, and UndefinedBehaviorSanitizer in the same compilation unit; sanitizer violations are caught inline without a separate crash-detection layer
- **Portable harness interface**: the `LLVMFuzzerTestOneInput` signature is the de facto standard for security harnesses; the same harness runs under libFuzzer, AFL++, and Honggfuzz
- **Dictionary support**: a structured token dictionary guides mutation toward valid syntax for formats such as JSON, HTML, SQL, and binary protocols
- **Corpus merge and minimize**: `--merge=1` deduplicates and minimizes a corpus to the smallest set that preserves all discovered edges; `--minimize_crash=1` shrinks a crashing input to its minimum reproducer
- **Fine-grained control flags**: `--max_total_time`, `--runs`, `--max_len`, `--jobs` allow precise control of fuzz sessions without wrapper scripts
- **Custom mutators**: implement `LLVMFuzzerCustomMutator` to guide mutation for highly structured inputs that random byte-level mutation explores poorly
- **Value profile mode**: `-use_value_profile=1` tracks per-comparison value pairs, helping the fuzzer break through equality checks and magic-byte guards faster

## Use Cases

- Fuzzing C and C++ library APIs and parsers where in-process speed matters
- Fuzzing Rust code via cargo-fuzz
- Writing portable harnesses that run unchanged under both libFuzzer and AFL++
- Integrating with OSS-Fuzz for continuous fuzzing of open-source projects
- Fuzzing targets with expensive initialization (initialize once, fuzz in the persistent loop)

## Installation & Setup

### Prerequisites

- `clang` version 6.0 or later (libFuzzer is bundled; no separate install needed)
- Linux or macOS (Windows support is partial)

### Verify availability

```bash
clang --version
# libFuzzer is included in any clang >= 6.0
# Confirm with:
echo "int LLVMFuzzerTestOneInput(const unsigned char *d, long unsigned int s){return 0;}" \
  | clang -fsanitize=fuzzer -x c - -o /tmp/fuzz_check && echo "libFuzzer available"
```

libFuzzer ships with the Clang distribution (no separate installation is required). On Debian/Ubuntu:

```bash
sudo apt-get install clang
```

On macOS with Homebrew:

```bash
brew install llvm
export PATH="$(brew --prefix llvm)/bin:$PATH"
```

## Usage Examples

### Basic Usage

Write a harness:

```c
// fuzz/fuzz_parse.c
#include <stdint.h>
#include <stddef.h>
#include "parser.h"

int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    if (size > 65536) return 0;  // guard against intentionally huge inputs
    parse_input(data, size);
    return 0;
}
```

Compile with fuzzer and AddressSanitizer:

```bash
clang -fsanitize=fuzzer,address,undefined \
  -o fuzz/fuzz_parse \
  fuzz/fuzz_parse.c src/parser.c \
  -I include/
```

Run:

```bash
mkdir -p corpus/
echo "valid_seed" > corpus/seed1
./fuzz/fuzz_parse corpus/
```

libFuzzer prints a status line every few seconds showing the number of runs, coverage edges hit, and corpus size. It saves crashing inputs in the current directory as `crash-<hash>`.

### Replay a Single Input

```bash
# Replay a corpus file or a crash — no fuzzing, just execute once
./fuzz/fuzz_parse corpus/seed1
./fuzz/fuzz_parse crash-abc123def456
```

### With MemorySanitizer

MemorySanitizer detects reads from uninitialized memory (bugs ASan does not catch):

```bash
# MSan requires a fully instrumented libc; use a prebuilt MSan runtime or build from source
clang -fsanitize=fuzzer,memory -fsanitize-memory-track-origins=2 \
  -o fuzz/fuzz_parse_msan \
  fuzz/fuzz_parse.c src/parser.c -I include/

./fuzz/fuzz_parse_msan corpus/
```

### Corpus Merge and Minimization

Merge multiple corpus directories and deduplicate to only the inputs that cover new edges:

```bash
./fuzz/fuzz_parse -merge=1 merged_corpus/ corpus/ extra_corpus/
```

Minimize a crashing input to its smallest reproducer:

```bash
./fuzz/fuzz_parse -minimize_crash=1 -runs=10000 crash-abc123def456
```

### Dictionary-Guided Fuzzing

For structured targets (JSON, SQL, HTTP), a dictionary of tokens dramatically accelerates exploration:

```
# dict/json.dict
"\""
":"
"{"
"}"
"["
"]"
"null"
"true"
"false"
```

```bash
./fuzz/fuzz_parse -dict=dict/json.dict corpus/
```

### Limiting Time and Runs

```bash
# Run for exactly 60 minutes
./fuzz/fuzz_parse -max_total_time=3600 corpus/

# Run for exactly one million iterations
./fuzz/fuzz_parse -runs=1000000 corpus/

# Limit input size to 1 KB
./fuzz/fuzz_parse -max_len=1024 corpus/
```

### Parallel Fuzzing

```bash
# Run 4 parallel jobs, each contributing to the same corpus
./fuzz/fuzz_parse -jobs=4 -workers=4 corpus/
```

Each worker writes its findings to `fuzz-N.log` and saves crashes in the working directory.

### Custom Mutator

For targets that require structurally valid input (e.g., a binary protocol with a checksum), implement a custom mutator to preserve validity during mutation:

```c
#include <stdint.h>
#include <stddef.h>
#include <string.h>

// Called by libFuzzer instead of its default mutator
size_t LLVMFuzzerCustomMutator(uint8_t *data, size_t size,
                                size_t max_size, unsigned int seed) {
    // Apply domain-specific mutation, e.g., flip a field, adjust a length
    // Return the new size
    if (size < 4) return size;
    data[seed % size] ^= (uint8_t)(seed >> 8);
    return size;
}
```

### Configuration Example

A complete build-and-run script for a CI fuzz job:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Build
clang -fsanitize=fuzzer,address,undefined \
  -fno-omit-frame-pointer -g \
  -o fuzz/fuzz_parse \
  fuzz/fuzz_parse.c src/parser.c -I include/

# Minimize seed corpus
mkdir -p corpus_min/
./fuzz/fuzz_parse -merge=1 corpus_min/ corpus/ 2>/dev/null || true

# Fuzz for 30 minutes
./fuzz/fuzz_parse \
  -max_total_time=1800 \
  -max_len=4096 \
  -print_final_stats=1 \
  corpus_min/

echo "Fuzzing complete - check working directory for crash-* files"
```

## Pros and Cons

### Advantages

- **No dependencies beyond clang**: libFuzzer ships with the compiler; no additional install, no daemon, no configuration files
- **Highest throughput for fast targets**: in-process execution removes fork overhead; well-suited to targets that initialize quickly and parse small inputs
- **Portable harness interface**: `LLVMFuzzerTestOneInput` harnesses run unmodified under AFL++, Honggfuzz, and OSS-Fuzz
- **Tight sanitizer integration**: ASan, MSan, and UBSan instrumentation compiles into the same binary, making sanitizer violations immediate and precise
- **Built-in corpus and crash management**: merge, minimize, and replay are first-class flags (no wrapper scripts needed)

### Limitations

- **In-process isolation**: a bug that corrupts the fuzzer's own state (e.g., a global destructor that crashes) can terminate the entire fuzzing session; AFL++'s forking model is more resilient to such bugs
- **Single-language focus**: libFuzzer targets C, C++, and Rust (via cargo-fuzz); fuzzing Python or JavaScript with it requires extra tooling
- **No out-of-the-box parallel coordination across machines**: multi-machine fuzzing requires manual corpus synchronization or an external orchestrator (OSS-Fuzz, ClusterFuzz)
- **Requires clang**: projects built exclusively with GCC need to add a clang build step; AFL++ supports GCC instrumentation natively

## Integration

### GitHub Actions (scheduled libFuzzer job)

```yaml
name: scheduled-libfuzzer

on:
  schedule:
    - cron: "0 2 * * 2"  # weekly, Tuesday 02:00 UTC

jobs:
  fuzz:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install clang
        run: sudo apt-get install -y clang

      - name: Build fuzz harness
        run: |
          clang -fsanitize=fuzzer,address,undefined \
            -fno-omit-frame-pointer -g \
            -o fuzz/fuzz_parse \
            fuzz/fuzz_parse.c src/parser.c -I include/

      - name: Restore corpus cache
        uses: actions/cache@v4
        with:
          path: corpus/
          key: libfuzzer-corpus-${{ github.sha }}
          restore-keys: libfuzzer-corpus-

      - name: Run fuzzer (45 minutes)
        run: |
          mkdir -p corpus/
          ./fuzz/fuzz_parse -max_total_time=2700 -print_final_stats=1 corpus/ \
            2>&1 | tee fuzz.log || true

      - name: Fail on crashes
        run: |
          crashes=$(ls crash-* 2>/dev/null | wc -l)
          if [ "$crashes" -gt 0 ]; then
            echo "$crashes crash(es) found"
            exit 1
          fi

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: libfuzzer-crashes
          path: crash-*
```

### Rust (cargo-fuzz)

cargo-fuzz wraps libFuzzer for Rust projects without any manual clang invocations:

```bash
cargo install cargo-fuzz
cargo fuzz add fuzz_parse                              # creates fuzz/fuzz_targets/fuzz_parse.rs
cargo fuzz run fuzz_parse                              # builds and starts fuzzing
cargo fuzz run fuzz_parse -- -max_total_time=3600
cargo fuzz tmin fuzz_parse <path/to/crash/artifact>   # minimize a crash to smallest reproducer
```

Generated harness template:

```rust
// fuzz/fuzz_targets/fuzz_parse.rs
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if let Ok(s) = std::str::from_utf8(data) {
        let _ = my_crate::parse(s);
    }
});
```

For structured inputs, use `arbitrary` to derive random but type-valid values:

```rust
use libfuzzer_sys::fuzz_target;
use arbitrary::Arbitrary;

#[derive(Arbitrary, Debug)]
struct FuzzInput {
    length: u16,
    payload: Vec<u8>,
}

fuzz_target!(|input: FuzzInput| {
    let _ = my_crate::process(input.length, &input.payload);
});
```

### OSS-Fuzz Integration

OSS-Fuzz runs libFuzzer-based harnesses continuously against open-source projects for free. A minimal integration requires:

```bash
# oss-fuzz/projects/my-project/build.sh
$CC $CFLAGS -c src/parser.c -o parser.o -I include/
$CC $CFLAGS $LIB_FUZZING_ENGINE fuzz/fuzz_parse.c parser.o \
  -o $OUT/fuzz_parse
cp -r corpus/ $OUT/fuzz_parse_seed_corpus.zip
```

OSS-Fuzz sets `$CC`, `$CFLAGS`, and `$LIB_FUZZING_ENGINE` automatically based on the chosen fuzzing engine (libFuzzer or AFL++).

## Alternatives

- **AFL++** - process-based fuzzer; more resilient to in-process state corruption; better for targets with heavy initialization cost; supports QEMU mode for binaries without source
- **Honggfuzz** - strong persistent-mode network fuzzing, hardware feedback (Intel PT) support
- **Atheris** - libFuzzer-based Python fuzzer from Google; re-uses `LLVMFuzzerTestOneInput` semantics at the Python layer

libFuzzer and AFL++ are often run **together**. AFL++'s `afl-clang-fast` can compile and run libFuzzer-compatible harnesses, giving the same harness two fuzzing engines with different mutation strategies and coverage feedback mechanisms.

## Resources

- [Official libFuzzer Documentation (LLVM)](https://llvm.org/docs/LibFuzzer.html)
- [libFuzzer Tutorial](https://github.com/google/fuzzing/blob/master/tutorial/libFuzzerTutorial.md)
- [cargo-fuzz (Rust)](https://rust-fuzz.github.io/book/cargo-fuzz.html)
- [OSS-Fuzz Integration Guide](https://google.github.io/oss-fuzz/getting-started/new-project-guide/)
- [The Fuzzing Book (free, online)](https://www.fuzzingbook.org/)
- [Google's Fuzzing Guidelines](https://github.com/google/fuzzing)

## License

Apache License 2.0 (part of the LLVM Project)

## Tags

`fuzzing` `coverage-guided` `libfuzzer` `llvm` `clang` `memory-safety` `security-testing` `c` `cpp` `rust`

---

**Contributed by:** @odraude23
**Last Updated:** 2026-06-17
