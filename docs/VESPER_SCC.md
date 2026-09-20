The Vesper Self-Compilation Specification

Version 0.1 (Draft)
Codename: "Ouroboros"
Goal: vsc written in Vesper, compiles Vesper

---

Table of Contents

1. Introduction
2. Why Self-Compile
3. Terminology: Stages
4. The Bootstrap Chain
5. Requirements on the Language
6. Requirements on the Toolchain
7. Bootstrap Process
8. The Fixpoint Property
9. Compiler-Specific Language Features
10. The Standard Library Problem
11. Performance Requirements
12. Verification and Trust
13. Reproducible Bootstrap
14. Handling Breaking Language Changes
15. Testing the Bootstrap
16. Risks and Mitigations
17. Timeline
18. Appendix A: Bootstrap Commands
19. Appendix B: Historical Precedents

---

1. Introduction

A compiler is self-hosting (or self-compiling) when it can compile its own source code. For Vesper, this means:

· vsc is written in Vesper, not in Go.
· The Vesper source of vsc is compiled by a previous version of vsc.
· The result is a new vsc that behaves identically to its predecessor.

Self-hosting is the most important milestone for any programming language. It proves that the language is expressive enough to build a real, complex system; that the toolchain is mature enough to handle it; and that the language is worth using for its own implementation.

This document specifies how Vesper reaches self-hosting, how the bootstrap process works, and how the resulting compiler is verified.

The project is codenamed Ouroboros — the snake that eats its own tail.

---

2. Why Self-Compile

2.1 The case for

Proof of maturity. A language that cannot express its own compiler is not ready for serious use. Self-hosting is the ultimate integration test: every feature of the language is exercised by a single, large, non-trivial program.

Dogfooding. The compiler team is forced to use the language every day. Every friction point, every awkward syntax, every missing feature becomes obvious and painful. This accelerates language design.

Contributor accessibility. Contributors no longer need to know Go to work on the Vesper compiler. If you know Vesper, you can improve vsc.

Performance. Vesper on ARC VM can be tuned for compiler workloads (heavy allocation, deep recursion, parallelism). A Go implementation is bounded by Go's performance characteristics.

Portability. Once vsc is written in Vesper, porting the compiler to a new platform only requires porting the ARC VM, not Go.

Independence. The Vesper project no longer depends on Go for its own tooling. The Go stage-0 compiler becomes a historical artifact.

2.2 The case against

Chicken-and-egg. Bootstrapping a compiler requires a working compiler. Historically, this has been solved by writing an initial implementation in another language (Go, in our case), then gradually replacing it.

Trust. When a compiler compiles itself, a malicious compiler could hide a backdoor in its own source, propagated to every future version (see Ken Thompson's Reflections on Trusting Trust). This is addressed in §12.

Maintenance burden. Two implementations (Go and Vesper) must be kept in sync during the transition, then one must be deprecated.

Bootstrap fragility. Any change to the language must be compilable by the previous version of itself. This constrains language evolution.

Despite these costs, self-hosting is not optional for a serious language. The plan below addresses each concern.

---

3. Terminology: Stages

Self-hosting follows the classic stage-N bootstrap pattern.

Stage 0 (S0) — Go Compiler

vsc written in Go. This is the initial implementation. It compiles Vesper source to ARC assembly. It is the seed from which everything else grows.

· Language: Go
· Input: .vsp files
· Output: .asm files, .arc binaries
· Status: Exists, maintained until S3 is stable.

Stage 1 (S1) — Vesper Compiler, Built by S0

vsc written in Vesper. Compiled by S0.

· Language: Vesper
· Input: .vsp files
· Output: .asm files, .arc binaries
· Built by: S0
· Status: Planned. Achieved when the Vesper implementation of vsc passes the same conformance suite as S0.

Stage 2 (S2) — Vesper Compiler, Built by S1

The same Vesper source as S1, but compiled by S1 instead of S0.

· Language: Vesper
· Input: .vsp files
· Output: .asm files, .arc binaries
· Built by: S1
· Status: Planned. Achieved when S1 can compile the S1 source into a working S2.

Stage 3 (S3) — Vesper Compiler, Built by S2

The same Vesper source as S1 and S2, but compiled by S2.

· Language: Vesper
· Input: .vsp files
· Output: .asm files, .arc binaries
· Built by: S2
· Status: Planned. When S3 behaves identically to S2, the fixpoint is reached, and the compiler is self-hosting.

Fixpoint

Once S2 = S3 (bit-identical, or behaviorally identical if bit-identical is not achievable due to signatures/timestamps), the compiler has reached its fixpoint. From this point forward, only S3 (or later stages) is needed to compile new versions of vsc.

---

4. The Bootstrap Chain

```
   ┌─────────────────┐
   │  vsc (in Go)    │  Stage 0
   │  S0             │
   └────────┬────────┘
            │ compiles the Vesper source of vsc
            ▼
   ┌─────────────────┐
   │  vsc (in Vesper)│  Stage 1
   │  S1             │
   └────────┬────────┘
            │ compiles the same Vesper source
            ▼
   ┌─────────────────┐
   │  vsc (in Vesper)│  Stage 2
   │  S2             │
   └────────┬────────┘
            │ compiles the same Vesper source
            ▼
   ┌─────────────────┐
   │  vsc (in Vesper)│  Stage 3
   │  S3             │
   └─────────────────┘
            │
            ▼
        Fixpoint: S2 ≡ S3
```

Why three Vesper stages? S1 is compiled by Go, so it inherits whatever quirks the Go compiler introduces (or misses). S2 is compiled by S1, so it reflects S1's understanding of the language. S3 is compiled by S2, and if S3 behaves identically to S2, then S2's understanding of the language is consistent with itself — the fixpoint.

Three stages are the minimum for confidence. Some compilers (Rust, Go) use more stages in practice or verify with third-party tools.

---

5. Requirements on the Language

For self-compilation to succeed, Vesper must support the following features. These are requirements, not wishes.

5.1 Core requirements

Feature Why needed
Records / structs Represent AST nodes, symbols, types
Sum types (enums) Represent variant AST nodes, errors, results
Pattern matching Dispatch over AST nodes
Recursion Traverse trees, resolve types
Tail-call optimization Deep recursive descent without stack overflow
Generic types Collections, maps, sets, generic algorithms
Interfaces Abstract over AST node kinds, symbol kinds
Closures Callbacks for visitors, type unification
Strings (UTF-8) Source text, identifiers, error messages
Lists, maps Symbol tables, error lists, module graphs
Optional values (Option<T>) Absence of type info, missing map keys
Result type (Result<T, E>) Error propagation without exceptions
? operator Clean error propagation through deep call chains
Modules Organize compiler into subsystems
Public/private visibility Encapsulate internal representation
File I/O Read .vsp files, write .asm files
Command-line argument parsing CLI behavior
Parallelism Compile modules in parallel
Channels Communicate between compiler threads
Deterministic hashing Map from identifiers to symbols
String formatting Error messages, dumps
Integer arithmetic (i32, i64, u32, u64) All internal computations
Floating-point (f64) Constant folding of f64 literals

5.2 Advanced requirements

Feature Why needed
First-class functions Pass algorithms as arguments
Method receivers Object-oriented-style code organization
Embedded structs Compose AST node types
Interface embeddings Extend interfaces
Type assertions Runtime type inspection for dynamic dispatch
Reflective-like access Not required; type assertion suffices
Constant expressions Compute lookup tables at compile time
Compile-time evaluation (const fn) Build symbol tables, hash tables at compile time
Weak references Avoid cycles in the AST → symbol graph
Arena allocation Efficient allocation of AST nodes
Interning Deduplicate strings, identifiers

5.3 What we do NOT need

Vesper has some features that self-compilation does not require, but they must still work:

· Concurrency is used for module parallelism and LSP, not for the core compiler algorithm.
· Macros (if ever added) would help reduce boilerplate, but are not required.
· Dependent types (if ever added) are not needed for a compiler.
· Refinement types (if ever added) are not needed.

5.4 Language stability

Before self-hosting can begin, the language must be frozen for a release cycle. No breaking changes to syntax or semantics may be introduced while the bootstrap is in progress. Small additive changes are permitted if they are backwards compatible.

---

6. Requirements on the Toolchain

6.1 Standard library

The compiler needs a substantial standard library:

Module Required by vsc
std.io Yes — file reading, writing, stdout
std.text Yes — string manipulation, formatting
std.collections Yes — hash maps, sets, vectors
std.fs Yes — directory traversal
std.path Yes — path joining, normalization
std.process Yes — spawning arc-as, arc-link
std.os Yes — environment variables, exit codes
std.time Only for logging timestamps
std.crypto Only for hashing (cache keys, arc-sign)
std.math Only for IEEE-754 constant folding
std.sync Yes — mutexes, for parallel compile
std.test Yes — the compiler must be tested

All of these modules must be complete and stable before bootstrap.

6.2 Build system

The bootstrap is orchestrated by a build script. It must:

1. Detect the current stage.
2. Invoke the correct compiler.
3. Assemble and link with arc-as, arc-link.
4. Verify the output.
5. Report progress.

The build script itself should eventually be written in Vesper.

6.3 Debug information

To debug the compiler, vsc must emit debug info for the .arc binaries it produces. This means:

· Line tables (source → ARC address).
· Symbol tables (Vesper name → ARC label).
· Type info for debugger display.

Without debug info, debugging the compiler is nearly impossible.

6.4 Deterministic output

The bootstrap requires bit-identical outputs at each stage. Any source of nondeterminism must be eliminated:

· Hash map iteration order → use sorted maps or insertion-ordered maps.
· File system ordering → sort by path.
· Timestamps → strip or use SOURCE_DATE_EPOCH.
· Pointer values → never embed in output.
· Random seeds → fixed.

6.5 Fast compilation

Compiler source is large. vsc must compile itself within minutes, not hours, on developer hardware. Target: < 5 minutes for a full bootstrap on a laptop, < 30 seconds for incremental.

---

7. Bootstrap Process

7.1 Phase 1: Go → Vesper (S0 → S1)

Duration: 12–18 months.

Goal: Rewrite vsc in Vesper, module by module, while keeping S0 as the compiler.

Process:

1. Port the lexer to Vesper. Compile it with S0. Verify output matches S0's lexer.
2. Port the parser. Same verification.
3. Port the resolver. Same verification.
4. Port the type checker. Same verification.
5. Port the HIR lowerer. Same verification.
6. Port the LIR lowerer. Same verification.
7. Port codegen. Same verification.
8. Port the CLI and driver. Same verification.
9. S0 now compiles the entire Vesper source of vsc.

Verification at each step: the S1 output must match the S0 output bit-for-bit on the entire conformance suite.

What if it doesn't match? The mismatch is a bug in either S0 or the Vesper port. Debug until they agree.

7.2 Phase 2: S1 → S2

Duration: 1 week.

Goal: Use S1 to compile itself.

Process:

1. S0 compiles the Vesper source → produces S1 binary.
2. S1 compiles the same source → produces S2 binary.
3. Compare S1 and S2 behavior on the conformance suite.
4. If identical, S2 is correct.

Why is this fast? By the time Phase 2 begins, S1 is already a fully functional compiler. Compiling itself is just another input.

Common failure mode: S1 has a bug that only manifests when compiling S1's own source. Debugging this is delicate — you cannot use S1 to debug S1. Fall back to S0 to compile a fixed S1, repeat.

7.3 Phase 3: S2 → S3, Fixpoint

Duration: 1 day.

Goal: Achieve the fixpoint.

Process:

1. S1 compiles the Vesper source → S2.
2. S2 compiles the same source → S3.
3. Compare S2 and S3 behavior, and if possible, byte-for-byte output.

Fixpoint condition: S2 and S3 are identical (byte-identical output files, or behaviorally identical given that signatures may introduce bytes). Once achieved, the compiler is self-hosting.

7.4 Phase 4: Deprecate S0

Duration: 6 months of overlap.

Goal: Remove the Go compiler from the critical path.

Process:

1. Announce S0 deprecation.
2. Freeze S0's source.
3. Continue maintaining S0 as a bootstrap seed.
4. Eventually publish S0 as a frozen bootstrap artifact — a single binary that can rebuild Vesper from scratch.

The frozen S0 binary is the "seed" from which all future Vesper versions grow. It is signed, published, and archived.

---

8. The Fixpoint Property

8.1 Definition

Let V be the source code of the Vesper compiler, and let C be a Vesper compiler.

The compiler is self-hosting if C(V) produces a compiler C' such that C' behaves identically to C. That is, C' compiles any program P to the same output as C.

8.2 Bit-identical fixpoint

The strongest form of fixpoint: C(V) == C'(V) byte-for-byte. Both compilers produce identical .arc binaries.

This is achievable if:

· The compiler is deterministic.
· Debug info embeds no timestamps.
· Signatures use reproducible keys (or are omitted during comparison).

8.3 Behavioral fixpoint

If bit-identical is not achievable (e.g., due to embedded paths or signatures), the fixpoint is behavioral: C and C' produce identical output on all programs in the conformance suite.

This is weaker but acceptable if the difference is documented and cannot hide a malicious payload.

8.4 Verifying the fixpoint

The bootstrap is verified by a script:

```
$ ./bootstrap.sh
Stage 0: build vsc in Go ...................... done
Stage 1: build vsc with S0 .................... done
Stage 2: build vsc with S1 .................... done
Stage 3: build vsc with S2 .................... done
Comparing S2 and S3 output ................... identical
Fixpoint reached.
```

If any stage fails or the comparison fails, the script exits with an error.

---

9. Compiler-Specific Language Features

The compiler needs features that ordinary user code does not. These are provided by the language as built-in or via attributes.

9.1 Compile-Time Evaluation (const fn)

The compiler uses lookup tables, hash functions, and small computations that should be done at compile time:

```vesper
const fn fnv1a(s: string) -> u32:
    var hash: u32 = 2166136261
    for i in 0..len(s):
        hash ^= u32(s[i])
        hash *= 16777619
    hash

const KEYWORD_IF = fnv1a("if")
const KEYWORD_ELSE = fnv1a("else")
# ...
```

Without const fn, these computations would run at every invocation.

9.2 Attributes

Attributes annotate declarations with compiler-specific information:

```vesper
@inline(always)
fn hot_path(...) -> ...:
    ...

@cold
fn error_path(...) -> ...:
    ...

@deprecated("use `new_fn` instead")
fn old_fn(...) -> ...:
    ...
```

Attributes needed by the compiler:

Attribute Purpose
@inline(always/never/hint) Control inlining
@cold / @hot Branch prediction hints
@deprecated(msg) Mark deprecated
@no_mangle Disable name mangling (FFI)
@repr(C) Use C-compatible layout
@packed No padding
@test Mark test functions
@bench Mark benchmark functions
@cfg(cond) Conditional compilation

9.3 Interned Strings

The compiler deals with millions of identical strings (identifiers, paths). Interning ensures each unique string is stored once:

```vesper
let id = intern("foo")     # returns a u32 ID
let s = unintern(id)       # returns the string
```

Interning is provided by std.intern, backed by a hash map.

9.4 Arenas

The AST is allocated in a bump arena and freed all at once:

```vesper
let arena = Arena()
let node = arena.alloc(Node(...))
# ... many nodes ...
arena.free()               # frees everything at once
```

Arenas avoid per-node GC overhead. std.arena provides them.

9.5 Unsafe, But Bounded

For performance-critical paths, the compiler may need to bypass bounds checks. This is done via unsafe blocks:

```vesper
unsafe:
    let v = xs[fast_index]
```

unsafe blocks:

· Are audited in code review.
· Are limited in scope (one expression or statement).
· Do not disable memory safety at the ARC level (a check is still performed, but the compiler is allowed to skip the bounds check when it can prove safety).
· Are logged in the build (the compiler reports the number of unsafe blocks).

9.6 Value vs Reference Semantics

The compiler distinguishes between Point (value) and Point referenced via (p: Point) (reference). Large AST nodes are always passed by reference to avoid copying.

---

10. The Standard Library Problem

Self-hosting requires the standard library to be complete and correct. But the standard library is itself written in Vesper. This is the same chicken-and-egg problem one level down.

10.1 Bootstrap of std

The standard library bootstraps as follows:

1. Core parts of std (primitives, strings, collections, io) are written in Vesper, but may call into ARC syscall directly.
2. Higher-level parts of std are written on top of the core parts.
3. The compiler uses only the core parts plus a few higher-level modules (std.fs, std.process, std.path).

The std library must be complete before bootstrap begins, because the compiler depends on it.

10.2 Avoiding Cycles

The compiler depends on std. If std ever depended on the compiler (e.g., for macros), we'd have a cycle. This is avoided by design:

· No macros that require compiler support.
· No compile-time reflection that requires the compiler.
· std is a library, not a compiler extension.

10.3 Testing std Independently

Before the compiler uses std, std must pass its own conformance suite. Otherwise a bug in std will manifest as a bug in the compiler, which is much harder to debug.

10.4 Frozen std During Bootstrap

While the bootstrap is in progress, std is frozen. Any change to std restarts the bootstrap verification.

---

11. Performance Requirements

A compiler that takes hours to compile itself is not practical. The bootstrap sets performance targets:

11.1 Compiler performance

Metric Target
Lexer throughput 100 MB/s
Parser throughput 50 MB/s
Type checker throughput 30 MB/s
HIR → LIR 40 MB/s
LIR → ASM 20 MB/s
Full compile of vsc source < 5 minutes
Incremental compile of vsc < 30 seconds

11.2 Runtime performance

The generated .arc for vsc must perform well:

· GC pauses < 10ms.
· Startup time < 100ms.
· Peak memory < 2 GB for the compiler.

11.3 JIT warmup

Because vsc is JIT-compiled by ARC VM, the first runs are slower. The bootstrap script must account for this:

· Use --jit-warmup to run the compiler on a small input first.
· Or measure both cold and warm performance separately.

---

12. Verification and Trust

12.1 Trusting Trust

Ken Thompson's 1984 paper Reflections on Trusting Trust describes a fundamental attack: a compiler can be modified to insert a backdoor into every program it compiles, including future versions of itself. Once the modified compiler is used to compile itself, the backdoor persists without appearing in the source.

This attack is real and has not been fully solved.

12.2 Mitigations for Vesper

Diverse double-compilation. Compile the same source with two independent compilers and compare the results. If they differ, at least one is buggy or malicious.

For Vesper:

· S0 is Go. Written by a different team, in a different language, on a different toolchain. Any malicious change to S1 must exist in both S0 and the Vesper source.
· S0 is auditable. Its source is public, small enough to review, and its binary is reproducible.
· S0 is frozen. Once the bootstrap is complete, S0 is archived and never modified.
· Third-party implementations. If the Vesper ecosystem produces an independent compiler (e.g., a Vesper-in-Rust implementation), it can be used for diverse double-compilation.

Reproducible builds. Every stage's output is reproducible from source. Any deviation is detected immediately.

SLSA. Every release of vsc includes SLSA provenance, signed by the build system.

Formal verification of the bootstrapped binary. Long-term, the goal is to formally verify that S3 implements the Vesper specification. This is beyond the scope of the initial bootstrap but is a goal.

12.3 What we cannot prevent

· A malicious S0 that inserts a backdoor into S1 that passes all tests and reproduces in S2, S3, ... forever.
· A malicious modification to the ARC VM (below the compiler).

These attacks require access to the build pipeline or the VM. Defense-in-depth (signed releases, reproducible builds, diverse implementations) raises the cost significantly but does not eliminate the risk.

---

13. Reproducible Bootstrap

13.1 Requirement

Anyone with:

· The Vesper source of vsc.
· A signed S0 binary.
· A conforming ARC VM.

...must be able to reproduce the entire bootstrap chain and arrive at byte-identical S1, S2, S3.

13.2 Sources of nondeterminism

Source Solution
Hash map iteration Sorted iteration or insertion-ordered maps
File system ordering Sort by path
Timestamps SOURCE_DATE_EPOCH or strip
Random seeds Fixed seed or seeded by input hash
Locale Force C locale
Environment Explicit, minimal env passing
Pointer values Never embed
Concurrency Deterministic scheduling in compiler; ordered output
Compiler version strings Sourced from a version file

13.3 Verification script

The bootstrap.sh script verifies reproducibility:

```bash
#!/bin/bash
set -euo pipefail

SOURCE_HASH=$(sha256sum -r vsc-src/*.vsp | sha256sum | cut -d' ' -f1)
echo "Source hash: $SOURCE_HASH"

build() {
    local stage=$1
    local compiler=$2
    local output=$3
    echo "Building $stage with $compiler..."
    $compiler build --target arc-vm-amd64 -o $output
    echo "$stage hash: $(sha256sum $output | cut -d' ' -f1)"
}

build S1 ./s0/vsc S1.arc
build S2 ./S1.arc S2.arc
build S3 ./S2.arc S3.arc

if cmp -s S2.arc S3.arc; then
    echo "Fixpoint reached."
    exit 0
else
    echo "Fixpoint NOT reached. S2 and S3 differ."
    exit 1
fi
```

13.4 Publishing the bootstrap

Every release of the compiler publishes:

· vsc-<version>-s0.arc — the frozen S0 seed.
· vsc-<version>-s1.arc, s2.arc, s3.arc — the three Vesper stages.
· bootstrap.sh — the script to reproduce.
· SHA256SUMS — checksums.
· SHA256SUMS.sig — signature by the release key.

---

14. Handling Breaking Language Changes

14.1 The problem

If the language changes, the compiler source must be updated to use the new syntax. But the current compiler cannot compile the new syntax until it is updated — a chicken-and-egg problem.

14.2 The two-version rule

Breaking changes are introduced in two steps:

Step 1 (minor release). The old syntax is deprecated but still accepted. The compiler is updated to use the new syntax internally. The compiler can now be compiled by the previous version, which understands both old and new.

Step 2 (major release). The old syntax is removed. At this point, the compiler's source is entirely in the new syntax, and the previous version (which still supports both) can compile it.

14.3 Example

Before:

```vesper
impl Point:
    fn length(self) -> f64: ...
```

Step 1. New syntax fn (p Point) length() -> f64 is introduced. Both impl and the new syntax are accepted.

Step 2. impl is removed from the language. Only the new syntax remains.

14.4 The "previous version" definition

For a breaking change in version X.Y.Z, the previous version is the latest release of the X.(Y-1).* line. The bootstrap path always exists from that version.

14.5 Long-term support (LTS)

Every major version has one LTS release, supported for at least 3 years. Breaking changes are only introduced in major versions. Minor and patch versions are backwards compatible.

---

15. Testing the Bootstrap

15.1 Bootstrap tests

Test Purpose
bootstrap-diff Compare output of S0, S1, S2, S3 on the conformance suite
bootstrap-fixpoint Verify S2 == S3 byte-for-byte
bootstrap-repro Reproduce the bootstrap on a clean machine
bootstrap-multi-host Reproduce on Linux, macOS, Windows
bootstrap-fast Verify incremental bootstrap under 30 seconds

15.2 Conformance suite

The same conformance suite that validates any Vesper compiler is used to validate each stage. A stage passes only if it passes all conformance tests.

15.3 Differential testing

Differential testing compares:

· S0 output vs S1 output.
· S1 output vs S2 output.
· S2 output vs S3 output.

Any difference is a bug in one of the stages.

15.4 Self-test

vsc compiles its own source in its test suite. If the compile fails, the build fails. This is not the full bootstrap (which involves stages), but it catches regressions in the compiler's ability to handle its own code.

15.5 CI integration

Every commit runs:

1. vsc test — unit tests.
2. vsc test-snapshots — golden file tests.
3. vsc self-test — compile vsc source with itself.
4. bootstrap.sh --short — S0 → S1 → S2 comparison.

Nightly CI runs the full bootstrap.

Weekly CI runs the bootstrap on macOS and Windows.

---

16. Risks and Mitigations

Risk Impact Mitigation
Compiler source too large to compile in reasonable time High Optimization passes; incremental compilation
Language missing a feature needed by the compiler High Feature freeze before bootstrap; use attributes if needed
S0 and S1 behave differently High Differential testing at every stage
Performance regression after self-hosting Medium Benchmark suite; profiling
New feature breaks bootstrap Medium Two-version rule; feature freeze
Standard library incomplete High Complete std before bootstrap; freeze during
Backdoor in S0 Critical Diverse double-compilation; audit; SLSA
Nondeterminism High Reproducible builds; CI verification
Debug info missing Medium Emit debug info from day one
Memory usage grows Medium Arena allocation; GC tuning

---

17. Timeline

The bootstrap is a multi-year process. A realistic timeline:

Phase Duration Milestone
Pre-bootstrap 12 months std complete; vsc in Go feature-complete
Lexer + parser in Vesper 3 months S0 can compile the Vesper lexer and parser
Resolver + type checker in Vesper 4 months S0 can compile the type checker
HIR/LIR/codegen in Vesper 6 months S0 can compile the entire compiler
CLI and driver in Vesper 2 months S0 → S1 complete
S1 self-compiles 1 month S1 → S2 complete
Fixpoint verification 1 month S2 ≡ S3
Deprecation of S0 6 months S0 frozen as bootstrap seed
Total ~2 years Vesper is self-hosting

After the initial bootstrap, ongoing work is:

· Maintaining the fixpoint.
· Improving performance.
· Adding features (with the two-version rule).
· Supporting new platforms.

---

Appendix A: Bootstrap Commands

The full bootstrap sequence, expressed as shell commands:

```bash
# Clone the repository
git clone https://github.com/vesperlang/vsc.git
cd vsc

# Fetch the S0 bootstrap binary
curl -o s0/vsc https://releases.vesper.dev/vsc-0.1.0-s0.arc
curl -o s0/vsc.sig https://releases.vesper.dev/vsc-0.1.0-s0.arc.sig
arc-verify --key release.pub --signature s0/vsc.sig s0/vsc

# Verify S0 works
arc-vm s0/vsc --version

# Bootstrap S1
arc-vm s0/vsc build --target arc-vm-amd64 -o S1.arc src/

# Bootstrap S2
arc-vm S1.arc build --target arc-vm-amd64 -o S2.arc src/

# Bootstrap S3
arc-vm S2.arc build --target arc-vm-amd64 -o S3.arc src/

# Verify fixpoint
cmp S2.arc S3.arc && echo "Fixpoint reached"

# Install
cp S3.arc ~/.local/bin/vsc
```

This is the sequence that a determined user can run to verify the bootstrap from scratch.

---

Appendix B: Historical Precedents

Self-hosting compilers have a rich history. The following are informative precedents for the Vesper project.

B.1 GCC

GCC is written in C (and later C++). It was originally bootstrapped from a C compiler (pcc, then a minimal C compiler). Today, GCC is bootstrapped from a small seed compiler written in C.

B.2 Go

Go's compiler was originally written in C. In 2015 (Go 1.5), the compiler was rewritten in Go, and the C compiler was retired. The bootstrap used Go 1.4 as the seed, which was itself built by the C compiler.

Lesson for Vesper: the Go team spent years on the transition. Careful staging pays off.

B.3 Rust

Rust's compiler is written in Rust. The bootstrap chain is: rustc-latest compiles rustc-next, which compiles rustc-next-next, ... until fixpoint. Rust uses a staged bootstrap with stages 0, 1, 2, 3 similar to this specification.

Lesson for Vesper: the fixpoint is the goal. Rust verifies it rigorously.

B.4 Haskell (GHC)

GHC is written in Haskell. It bootstraps from a previous GHC. The first GHC was written in Lazy ML, then self-hosted.

Lesson for Vesper: language features (laziness, type classes) had to be carefully designed to support self-hosting.

B.5 Swift

Swift's compiler is written in C++ (and Swift). Swift uses a bootstrap for the Swift parts of its toolchain, with the C++ parts serving as the seed.

Lesson for Vesper: some parts of the compiler may remain in the seed language for a long time.

B.6 Self (language)

The Self programming language was designed to be self-hosting from the start. Its first implementation was in Smalltalk, then rewritten in Self.

Lesson for Vesper: designing for self-hosting from the beginning makes the transition smoother.

B.7 Oberon

The Oberon compiler and operating system were self-hosting. The compiler was written in Oberon, compiled by a previous version, and the entire system could be rebuilt from source.

Lesson for Vesper: self-hosting extends to the entire toolchain, not just the compiler. Consider the LSP, formatter, and package manager as candidates for self-hosting too.

B.8 Summary of lessons

1. Design for self-hosting early. Features that seem optional become essential when you use the language to build its own tooling.
2. Stage the bootstrap carefully. The three-stage approach is standard; more stages add confidence but cost time.
3. Verify the fixpoint. Without byte-identical fixpoint, you cannot be sure the bootstrap is correct.
4. Freeze the language. No breaking changes during bootstrap.
5. Build the standard library first. The compiler depends on it.
6. Invest in tooling. Debug info, profiling, and reproducible builds are essential.
7. Plan for the long haul. Self-hosting takes years, not months.

---

End of specification.

---

Ouroboros — the snake that eats its own tail — is the oldest symbol of cyclicality and self-reference. The Vesper compiler, compiling itself, is the modern incarnation: a language proving itself by building its own future.

🐍
