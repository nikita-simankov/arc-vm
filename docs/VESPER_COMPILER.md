The Vesper Compiler Specification

Version 0.1 (Draft)
Compiler binary: vsc
Target: ARC Virtual Machine
Language version: Vesper 0.1

---

Table of Contents

1. Introduction
2. Invocation
3. Command-Line Interface
4. Project Layout
5. Compilation Pipeline
6. Lexical Analysis
7. Parsing
8. Name Resolution
9. Type Checking
10. Desugaring and Lowering
11. Intermediate Representations
12. Code Generation
13. Optimization Passes
14. Assembly Emission
15. Assembly and Linking
16. Incremental Compilation
17. Determinism and Reproducibility
18. Diagnostics
19. Exit Codes
20. Configuration Files
21. Embedding as a Library
22. Performance Requirements
23. Cross-Compilation
24. Testing the Compiler
25. Appendix A: Command Reference
26. Appendix B: Diagnostic Codes
27. Appendix C: IR Formats

---

1. Introduction

vsc is the reference compiler for the Vesper programming language. It reads Vesper source files (.vsp) and produces ARC assembly (.asm) or, via the ARC assembler, ARC binaries (.arc) ready to be loaded by the ARC VM.

The compiler is designed to be:

· Fast. A medium-sized project (10 000 lines) compiles in under one second on commodity hardware.
· Predictable. Identical inputs produce bit-identical outputs, regardless of machine, time, or environment.
· Informative. Diagnostics are precise, actionable, and reference the language specification.
· Modular. Every stage is exposed as a library API for tooling (LSP, formatter, linter, REPL).
· Safe. The compiler never executes user code. It is a pure function from source to artifact.

This document is normative: any compiler claiming to be a conforming Vesper compiler must implement the behavior described here. Additional behavior is allowed but must not contradict this specification.

---

2. Invocation

2.1 Forms

vsc accepts three invocation forms:

```
vsc <command> [flags] [arguments]
vsc [flags] <file.vsp>
vsc --version | --help
```

The second form is a shortcut for vsc build <file.vsp>.

2.2 Commands

Command Purpose
build Compile a project or a single file
run Compile and immediately run on ARC VM
check Parse and type-check without producing output
test Compile and run tests
bench Compile and run benchmarks
fmt (delegates to vsfmt)
doc (delegates to vsdoc)
clean Remove build artifacts
init Create a new project (delegates to vspm)
version Print version and exit
help Print usage information

2.3 Examples

```
vsc build                          # build the current project
vsc build src/main.vsp             # build a single file
vsc run src/main.vsp               # build and run
vsc check --workspace              # type-check all files
vsc build --emit asm               # emit .asm in addition to .arc
vsc build --emit ast,hir,lir       # emit all IRs
vsc build --target arc-vm-arm64    # cross-compile
vsc build --opt-level 3            # max optimization
vsc build --debug                  # include debug info
vsc test --filter "user_"          # run tests matching a pattern
```

---

3. Command-Line Interface

3.1 Global Flags

Flag Description Default
--help, -h Print help and exit —
--version, -V Print version and exit —
--color <mode> auto, always, never auto
--verbose, -v Increase verbosity (repeatable) 0
--quiet, -q Suppress non-error output false
--json Machine-readable output (diagnostics, progress) false
--config <file> Use a specific config file auto-detect
--no-config Ignore config files false

3.2 build Flags

Flag Description Default
--output, -o <path> Output file or directory build/
--emit <targets> Comma-separated: arc, asm, ast, hir, lir, tokens, symtab, deps arc
--opt-level <n> 0–3, s (size), z (min size) 1
--debug Include debug info in .arc false
--strip Strip debug info and symbols false
--target <triple> Target triple (see §23) host
--features <list> Enable optional features per project
--no-default-features Disable default features false
--incremental Enable incremental compilation true
--no-incremental Disable incremental compilation false
--jobs <n> Parallel jobs CPU count
--keep-temp Keep temporary files false
--deny <lint> Turn warnings into errors for the given lint —
--allow <lint> Allow the given lint —
--forbid <lint> Forbid the given lint (error, not overridable) —
--warn <lint> Enable a warning —

3.3 check Flags

check supports all build flags that affect analysis (all except --output, --emit arc/asm, --strip).

3.4 run Flags

run supports all build flags plus VM flags after --:

```
vsc run src/main.vsp -- --stack-size 1M --gc-threshold 100M arg1 arg2
```

3.5 Environment Variables

Variable Purpose
VSC_CONFIG Path to a config file
VSC_CACHE_DIR Cache directory (default ~/.cache/vsc)
VSC_TMPDIR Temporary directory
VSC_JOBS Default parallelism
VSC_COLOR Default color mode
VSC_LOG Log filter (e.g., vsc=debug,codegen=trace)

Flags override environment variables. Environment variables override config files.

---

4. Project Layout

4.1 Recognized Files

File Purpose
vsp.toml Project manifest
vsp.lock Locked dependency versions
vsc.toml Compiler configuration
.vsfmt.toml Formatter config (used by vsfmt)
.vslint.toml Linter config (used by vslint)
*.vsp Source files
*.test.vsp Test files
*.bench.vsp Benchmark files

4.2 Directory Structure

A canonical Vesper project:

```
my-project/
├── vsp.toml
├── vsp.lock
├── vsc.toml
├── src/
│   ├── main.vsp
│   ├── lib.vsp
│   └── geometry/
│       ├── point.vsp
│       └── vector.vsp
├── tests/
│   └── integration.test.vsp
├── benches/
│   └── perf.bench.vsp
├── examples/
│   └── basic.vsp
└── build/           ← generated
    ├── main.arc
    ├── main.asm
    └── .vsc-cache/
```

4.3 Module Discovery

A file's module path is derived from its path relative to src/:

· src/main.vsp → module main
· src/geometry/point.vsp → module geometry.point

The module declaration in the file must match this path. A mismatch is a compile error (VSP-E100).

4.4 Workspaces

A workspace is a set of crates sharing a build cache. Defined in vsp.toml:

```toml
[workspace]
members = ["core", "cli", "server"]
```

Each member has its own vsp.toml. The workspace root has a vsp.toml with only a [workspace] table.

---

5. Compilation Pipeline

5.1 Stages

```
.vsp source
    │
    ▼
[ Lexer ]───────► token stream
    │
    ▼
[ Parser ]──────► AST
    │
    ▼
[ Name Resolution ]───► resolved AST
    │
    ▼
[ Type Checker ]──────► typed AST
    │
    ▼
[ Desugar / Lower ]───► HIR
    │
    ▼
[ HIR Optimizer ]─────► optimized HIR
    │
    ▼
[ LIR Lowering ]──────► LIR
    │
    ▼
[ LIR Optimizer ]─────► optimized LIR
    │
    ▼
[ Codegen ]───────────► ARC assembly
    │
    ▼
.asm
    │
    ▼
[ arc-as ]────────────► .arc object
    │
    ▼
[ arc-link ]──────────► .arc executable
    │
    ▼
.arc (ready for arc-vm)
```

5.2 Stage Contracts

Each stage must:

· Consume only the output of the previous stage (no peeking at source).
· Be deterministic.
· Report diagnostics with accurate spans.
· Support early exit on errors (fail-fast for fatal errors, continue for recoverable ones).

5.3 Compilation Unit

A compilation unit is one module (one .vsp file) plus its transitive dependency graph. Each unit is compiled independently and cached.

---

6. Lexical Analysis

6.1 Input

The lexer reads a UTF-8 byte stream (the source file) and produces a stream of tokens with spans.

6.2 Token Kinds

· Identifier
· Keyword (see Vesper spec §4.4)
· IntegerLiteral, FloatLiteral, StringLiteral, RuneLiteral, AtomLiteral
· BooleanLiteral (true, false), NoneLiteral
· Operator (see Vesper spec §4.5)
· Punctuation
· Indent, Dedent, Newline (synthetic tokens)
· EOF

6.3 Indentation Handling

The lexer maintains an indentation stack:

· On a new line, compute leading whitespace.
· If greater than top of stack: push, emit Indent.
· If equal: emit Newline.
· If less: pop until match, emit Dedent for each pop.
· Blank lines and comment-only lines are ignored for indentation.

Tabs in leading whitespace are illegal (VSP-E110). Mixing tabs and spaces anywhere in leading whitespace is illegal (VSP-E111).

6.4 String Interpolation

"Hello, {name}!" is lexed into a sequence:

· StringLiteral("Hello, ")
· InterpolationStart
· Identifier("name")
· InterpolationEnd
· StringLiteral("!")

The parser assembles these into a single InterpolatedString node.

6.5 Line Continuation

A statement may be split across multiple physical lines if:

· The line ends inside an open delimiter ((, [, {).
· The line ends with a binary operator.
· The line ends with \ (explicit continuation — discouraged).

6.6 Lexer Diagnostics

Code Message
VSP-E100 Invalid character
VSP-E101 Unterminated string literal
VSP-E102 Unterminated block comment
VSP-E103 Invalid escape sequence
VSP-E104 Invalid numeric literal
VSP-E110 Tab in indentation
VSP-E111 Mixed tabs and spaces

The lexer continues after a diagnostic to collect more errors where possible.

---

7. Parsing

7.1 Grammar

The parser implements the grammar in the Vesper Language Specification, Appendix A.

7.2 Parser Type

The parser is recursive descent with a Pratt-style expression parser for operator precedence and binding.

7.3 Error Recovery

· On a syntax error, the parser inserts an ErrorNode into the AST and skips tokens until a synchronization point (Newline, Dedent, end, else, elif, catch, EOF).
· Parse errors do not halt compilation; the rest of the file is parsed.
· A file with syntax errors produces no executable code (compilation halts after parsing).

7.4 AST

The AST preserves source structure exactly:

· Every node has a Span (byte range) and a Position (line, column).
· Nodes carry no type information at this stage.
· Comments are attached to the nearest following declaration as trivia.

7.5 AST Dumping

vsc build --emit ast writes a human-readable AST to <output>/<module>.ast:

```
Module(main)
├── Import(std.io)
├── Struct(Point) [line 5-8]
│   ├── Field(x: f64) [line 6]
│   └── Field(y: f64) [line 7]
├── Method(length) [line 10-11]
│   ├── Receiver(p: Point)
│   └── Body
│       └── Call(sqrt)
│           └── Binary(+)
│               ├── Binary(*)
│               │   ├── Field(p.x)
│               │   └── Field(p.x)
│               └── Binary(*)
│                   ├── Field(p.y)
│                   └── Field(p.y)
└── Function(main) [line 15-20]
    └── Body
        └── Call(print)
```

7.6 Parser Diagnostics

Code Message
VSP-E200 Expected token X, found Y
VSP-E201 Unexpected end of file
VSP-E202 Unexpected indentation
VSP-E203 Expected : after block introducer
VSP-E204 Expected block
VSP-E205 Invalid pattern
VSP-E206 Invalid type expression
VSP-E207 Reserved keyword used as identifier

---

8. Name Resolution

8.1 Purpose

The resolver binds every identifier to its declaration. After resolution, no identifier is ambiguous.

8.2 Scopes

The resolver builds a scope tree:

· Module scope: top-level declarations.
· Function scope: parameters, local variables.
· Block scope: nested blocks (if, for, match, try, etc.).
· Type scope: struct fields, enum variants, interface methods.

8.3 Resolution Rules

1. Local shadowing: a local variable shadows an outer one of the same name. Shadowing of a pub top-level name is a warning (VSP-W003).
2. Import resolution: import foo.bar.{baz} binds baz in module scope.
3. Method resolution: x.foo() first searches methods on x's type, then on embedded types, then on interfaces x satisfies.
4. Field resolution: x.field searches struct fields, then promoted embedded fields.
5. Method receiver: self is not used in Vesper; instead, the receiver is named explicitly in the method declaration. The resolver binds the receiver name in the method's scope.

8.4 Prelude

Every module implicitly imports the Vesper prelude:

· Built-in types: i8–u64, f32, f64, bool, string, Any, Option, Result, Atom.
· Built-in functions: len, cap, print, panic, assert, min, max, abs, sqrt, self, send, recv, close, append, concat, keys, values, get, has, int_to_string, string_to_int, char_at, substring.
· Built-in constructors: Some, None, Ok, Err.
· Built-in interfaces: Any, Comparable, Display, Hash, Iterator.

Prelude items can be shadowed by user declarations (with a warning VSP-W004).

8.5 Diagnostics

Code Message
VSP-E300 Undefined variable x
VSP-E301 Undefined function f
VSP-E302 Undefined type T
VSP-E303 Undefined module m
VSP-E304 Symbol x is not exported from module m
VSP-E305 Cyclic import
VSP-E306 Module path m does not match file path
VSP-E307 Duplicate definition of x
VSP-W003 Shadowing outer binding x
VSP-W004 Shadowing prelude item x

---

9. Type Checking

9.1 Purpose

The type checker assigns a type to every expression and verifies that all operations are valid for their operand types.

9.2 Type System

Vesper uses a Hindley–Milner type system extended with:

· Subtyping through interfaces (structural).
· Row polymorphism for structs (embedding).
· Effect tracking for concurrency (which functions may spawn, send, receive).
· Refinement types for numeric ranges (optional, off by default).

9.3 Algorithm

Type inference uses Algorithm W with the following extensions:

· Local type inference: function bodies are inferred; function signatures must be explicit unless the function is local.
· Bidirectional checking: for constructs where full inference is expensive (e.g., struct literals, method calls), the checker alternates between synthesis (produce a type) and checking (verify against a known type).
· Union-find: for unification of type variables.
· Occurs check: prevents infinite types.

9.4 Type Variables

A type variable is created when:

· The type of a let binding is not annotated and the initializer is a generic function call.
· A function without a return type annotation is called.
· A field is accessed on a value whose type is not yet known.

Unsolved type variables at the end of a function are errors (VSP-E400).

9.5 Generic Instantiation

Generic functions are monomorphized. The instantiation algorithm:

1. Collect all call sites of a generic function.
2. For each unique combination of type arguments, create a specialized copy.
3. Replace the generic call with a call to the specialized copy.

Monomorphization happens during lowering (§10), not during type checking.

9.6 Trait/Interface Checking

An interface is satisfied by a type T if T's method set includes all of I's methods. The method sets are computed from:

· Methods declared directly on T.
· Methods promoted from embedded fields.
· Methods on T's pointer type (for reference receivers).

9.7 Type Checking of Patterns

A pattern is checked against the type of its scrutinee:

· Literal patterns match values of the literal's type.
· Identifier patterns bind the scrutinee's type.
· Tuple patterns destructure tuples and require matching arity.
· Struct patterns require all named fields to exist.
· Enum patterns require the variant to exist and the payload to match.
· Type patterns require the scrutinee to be an interface type.

9.8 Exhaustiveness

match expressions must be exhaustive. The checker computes the set of patterns and reports missing cases (VSP-E008).

9.9 Diagnostics

Code Message
VSP-E400 Cannot infer type
VSP-E401 Type mismatch: expected T, found U
VSP-E402 Not a function
VSP-E403 Wrong number of arguments
VSP-E404 Missing named argument x
VSP-E405 Unknown named argument x
VSP-E406 Method m not found on type T
VSP-E407 Field f not found on type T
VSP-E408 Interface I not satisfied by T
VSP-E409 Immutable binding x cannot be reassigned
VSP-E410 Cannot move from x
VSP-E411 Use of moved value x
VSP-E412 Non-exhaustive match
VSP-E413 Unreachable pattern
VSP-E414 ? used in function returning non-Result
VSP-E415 Generic argument T does not satisfy bound I
VSP-E416 Cyclic type
VSP-W005 Lossy numeric conversion
VSP-W006 Unused variable x
VSP-W007 Unused import m
VSP-W008 Unused function f
VSP-W009 Deprecated API
VSP-W010 Unreachable code after return/break/continue

---

10. Desugaring and Lowering

10.1 Purpose

The typed AST is desugared and lowered to HIR (High-level IR). This stage removes syntactic sugar and makes structure explicit.

10.2 Desugarings

Construct Desugars to
x += y x = x + y
a < b < c a < b && b < c
String interpolation "a{b}c" concat(concat("a", to_string(b)), "c")
Pipe x |> f(y) f(x, y)
Comprehension [f(x) for x in xs if p(x)] map(filter(xs, p), f)
match with patterns chain of if-elif-else with bindings
for x in xs iterator protocol
if cond: a else: b (expression) cond ? a : b (SSA form)
defer f() push to deferred list; executed on return
try/catch ARC exception handler registration
spawn f() thread creation
ch <- v channel send
v = <- ch channel receive

10.3 Monomorphization

Generic functions and types are specialized at this stage:

· Each instantiation of fn foo<T> with a specific T produces a specialized copy foo_<T>.
· Shared type-erased implementations are used for Any-based generic calls.

10.4 Closure Conversion

Lambdas and closures are converted to explicit records:

```
fn(x) -> x + n
```

becomes

```
struct Closure_1 { n: i32 }
fn closure_1_call(env: Closure_1, x: i32) -> i32:
    x + env.n
```

The closure is allocated in the heap; the environment is captured by value.

10.5 HIR Representation

HIR is a tree-structured IR with:

· Explicit bindings: every temporary is named.
· Explicit control flow: if, match, loop are preserved.
· Explicit types: every expression has a type.
· Source spans: preserved for diagnostics.
· No sugar: all syntactic conveniences are expanded.

---

11. Intermediate Representations

11.1 Overview

The compiler uses three IRs:

IR Purpose Level
AST Parsed source High
HIR Desugared, type-checked High
LIR SSA-based, optimized Low

11.2 AST

The AST is the direct result of parsing. It mirrors source structure. See §7.4.

11.3 HIR

HIR is a tree with explicit bindings and control flow. Designed for optimization and monomorphization.

HIR example:

```
fn add(a: i32, b: i32) -> i32:
    return + (a: i32) (b: i32)
```

Dump format:

```
fn add(a: i32, b: i32) -> i32:
  block:
    t0 = add(a, b)
    ret t0
```

11.4 LIR

LIR is a SSA-based IR with basic blocks and explicit control-flow graph. Designed for optimization and code generation.

LIR example:

```
fn add:
  entry:
    %0: i32 = param 0
    %1: i32 = param 1
    %2: i32 = add %0, %1
    ret %2
```

LIR invariants:

· Every value is defined exactly once.
· Every use is dominated by its definition.
· Every basic block ends with a terminator (br, br_cond, switch, ret, unreachable).
· Every value has an explicit type.

11.5 HIR → LIR Lowering

The lowering pass:

1. Converts HIR expressions to LIR SSA values.
2. Creates basic blocks for control flow.
3. Inserts phi nodes at block merges.
4. Adds explicit stack slots for spilled values.

11.6 SSA Construction

SSA construction uses the sealed blocks algorithm:

1. Traverse HIR in dominance order.
2. For each variable, maintain a current definition.
3. At block merges, insert phi nodes.
4. Use the standard dominance frontier algorithm to place phis.

11.7 Dumps

vsc build --emit hir,lir writes human-readable dumps:

· <module>.hir — HIR tree.
· <module>.lir — LIR with basic blocks.

Dumps are stable across runs (same input → same dump).

---

12. Code Generation

12.1 Purpose

Codegen translates LIR to ARC assembly. It is the only stage that knows about ARC registers, stack layout, and calling convention.

12.2 Register Allocation

The allocator uses linear scan with the following strategy:

1. Compute live intervals for each SSA value.
2. Order intervals by start point.
3. Allocate registers greedily; spill when necessary.
4. Use r0–r3 for arguments and returns.
5. Use r4–r11 for locals.
6. Use r12–r13 for scratch.
7. Spill to the stack when all callee-saved registers are busy.

12.3 Calling Convention

Parameters:

· First 4 parameters in r0–r3.
· Additional parameters pushed onto the data stack in reverse order.
· Return value in r0 (or r1:r0 for 64-bit).

Callee-saved:

· r4–r11 must be preserved.
· Method receivers occupy the first parameter slot.

12.4 Stack Frame Layout

```
+-------------------+  ← csp (top of call stack)
| saved r11         |
| saved r10         |
| ...               |
| saved r4          |
| local 0 (16 B)    |
| local 1 (8 B)     |
| ...               |
+-------------------+  ← fp
```

Locals are accessed via [fp - offset].

12.5 Instruction Selection

Each LIR operation maps to one or more ARC instructions:

LIR ARC
add i32 a, b add r0, r1, r2
add i64 a, b add64 r1:r0, r3:r2, r5:r4
mul f64 a, b fmul r1:r0, r3:r2, r5:r4
cmp i32 a, b cmp r1, r2
br_cond c, t, f cmp c, #0; jeq .t; jmp .f
phi resolved by register allocation
call f, args mov r0, a0; ...; call f

12.6 Method Calls

· Static dispatch: direct call point_length when the receiver type is known.
· Dynamic dispatch (interface): load vtable, load method pointer, call rN.

12.7 Constants

· Small integers → mov rN, #imm.
· Large integers → mov + or sequence.
· Floating-point → loaded from .rodata.
· Strings → pointer into .rodata.

12.8 Heap Allocation

· alloc(size) → mov r0, size; mov r3, #SYS_alloc; syscall.
· New objects are zero-initialized by ARC's alloc.
· Field initialization happens after allocation, via store.

12.9 Diagnostics

Code Message
VSP-E500 Too many registers to allocate
VSP-E501 Unsupported operation on target
VSP-E502 Internal: unknown LIR node

Errors at this stage usually indicate a compiler bug and should be reported.

---

13. Optimization Passes

13.1 Pass Order

Optimizations are applied in this order:

1. HIR passes:
   · Constant folding
   · Inline trivial functions
   · Dead code elimination
   · Loop-invariant code motion (limited)
2. LIR passes:
   · SSA construction
   · Copy propagation
   · Constant propagation
   · Common subexpression elimination
   · Dead code elimination
   · Loop-invariant code motion
   · Strength reduction
   · Tail-call optimization
   · Inlining
   · SSA destruction
3. Machine passes:
   · Register allocation
   · Peephole optimization
   · Instruction scheduling (limited)

13.2 Optimization Levels

Level Passes
0 None (debug-friendly)
1 Constant folding, copy propagation, DCE, tail-call (default)
2 + CSE, LICM, strength reduction, inlining (small)
3 + aggressive inlining, vectorization (future)
s Optimize for size
z Optimize for minimum size

13.3 Inlining Heuristics

A function is inlined if:

· Its body is under 20 instructions.
· It is called fewer than 5 times.
· It is not recursive.
· --opt-level >= 2, or it is marked #[inline].

13.4 Tail-Call Optimization

A call is a tail call if:

· It is the last operation before ret.
· The caller's frame is not needed after the call.
· The callee's return type matches the caller's.

Tail calls are compiled to jmp instead of call + ret.

13.5 Disabling Optimizations

Individual passes can be disabled for debugging:

```
vsc build --no-opt inline --no-opt dce
```

13.6 Correctness Requirements

Every optimization must preserve observable behavior. The compiler must not change:

· Order of side effects.
· Results of arithmetic (except allowed reassociations).
· Panic/exception behavior.
· Memory safety.

Optimizations that violate these are compiler bugs.

---

14. Assembly Emission

14.1 Output Format

vsc emits ARC assembly in the format expected by arc-as (see ARC specification). The output is:

· Text, UTF-8, \n line endings.
· One instruction per line.
· No tabs; spaces only.
· Deterministic ordering of sections and symbols.

14.2 Symbol Naming

Vesper symbols map to ARC labels with a mangling scheme:

Vesper ARC label
fn foo() foo
fn foo.bar() in module a.b a_b_foo_bar (dotted module paths are joined with _)
fn (p Point) length() Point_length
Generic fn id<T> instantiated with i32 id_i32
Lambda in function f f__closure_N
Static string .Lstr_N
Static f64 literal .Lf64_N

14.3 Sections

Generated assembly has exactly three sections:

· .code — functions.
· .rodata — string literals, floating-point literals, vtables.
· .data — global mutable variables.

14.4 Example

Vesper:

```vesper
fn add(a: i32, b: i32) -> i32:
    a + b

fn main():
    print(add(2, 3))
```

ARC assembly:

```asm
.code
.global _start

_start:
    enter #0
    call main
    mov  r0, #0
    mov  r3, #SYS_exit
    syscall

add:
    enter #0
    add   r0, r0, r1
    leave
    ret

main:
    enter #0
    mov   r0, #2
    mov   r1, #3
    call  add
    mov   r1, r0
    mov   r0, #0
    call  print_i32
    leave
    ret

.rodata
str_0:  .string "\n"

.const SYS_exit,  0
.const SYS_write, 1
```

14.5 Assembly Comment Preservation

Assembly comments are generated from source spans to aid debugging:

```asm
add:
    enter #0
    ; src/math.vsp:3:8
    add   r0, r0, r1
    ; src/math.vsp:3:13
    leave
    ret
```

Debug comments are omitted when --opt-level >= 2 or --strip.

---

15. Assembly and Linking

15.1 vsc build Pipeline

By default, vsc build produces a .arc executable by:

1. Emitting .asm for each module.
2. Invoking arc-as on each .asm to produce .arc objects.
3. Invoking arc-link to combine objects into an executable.
4. Invoking arc-sign to sign the executable (if a signing key is configured).

15.2 Tool Discovery

vsc finds arc-as, arc-link, and arc-sign via:

1. --arc-as <path>, --arc-link <path>, --arc-sign <path> flags.
2. $VSC_ARC_AS, etc., environment variables.
3. Config file vsc.toml (arc.tools.as.path, etc.).
4. $PATH.

15.3 Linking Modes

Mode Flag Behavior
Static (default) — All dependencies compiled into one .arc
Shared --shared Produce a .so-like shared ARC object
Object-only --emit obj Stop after arc-as, do not link

15.4 Signing

If --sign <key-file> is set, vsc invokes arc-sign after linking. The key file is an Ed25519 private key in PEM format.

In development mode (--no-sign or no key), the .arc is unsigned and can be loaded by arc-vm --allow-unsigned.

---

16. Incremental Compilation

16.1 Goal

Recompiling a small edit must be proportional to the size of the change, not the project.

16.2 Cache Keys

Each compilation unit's cache key is the hash of:

· The source file's contents.
· The public interface of every imported module (types, functions, constants).
· The compiler version and flags.
· The target triple.

16.3 Cache Storage

The cache lives in <output>/.vsc-cache/:

```
.vsc-cache/
├── index.json           # maps cache keys to artifacts
├── <hash>.hir           # HIR dumps (optional)
├── <hash>.lir           # LIR dumps (optional)
├── <hash>.asm           # ARC assembly
└── <hash>.deps          # dependency graph
```

16.4 Invalidation

A unit is recompiled if:

· Its source changed.
· The public interface of any direct import changed.
· The compiler version or flags changed.

Transitive imports do not invalidate directly; only interface changes propagate.

16.5 Interface Stability

The public interface of a module consists of:

· Exported types and their structure (fields, methods).
· Exported functions' signatures.
· Exported constants' values and types.

Changing a private function does not invalidate dependents.

16.6 Cache Format

Cache entries are binary-encoded. The format is versioned; a version mismatch discards all cache entries.

16.7 Disabling

--no-incremental disables caching. Useful for release builds.

---

17. Determinism and Reproducibility

17.1 Requirements

Given identical inputs (source files, dependencies, compiler version, flags), vsc must produce bit-identical output:

· .asm files are byte-for-byte identical.
· .arc files are byte-for-byte identical (signatures aside).
· IR dumps are identical.

17.2 Sources of Nondeterminism to Avoid

· Hash map iteration order.
· File system ordering.
· Thread scheduling.
· Time and date.
· Random number generators.
· Pointer values.
· Environment variables.

The compiler must explicitly sort all collections before output.

17.3 Timestamps

Generated .arc files contain no timestamps. If a timestamp is required (for example, in a build manifest), it comes from a reproducible source (SOURCE_DATE_EPOCH).

17.4 Verification

CI must run:

1. vsc build twice on the same input.
2. Compare outputs with cmp.
3. Fail on mismatch.

17.5 Cross-Platform Reproducibility

vsc must produce identical output on Linux, macOS, and Windows for the same input, modulo platform-specific paths embedded in debug info (which are stripped in release builds).

---

18. Diagnostics

18.1 Human-Readable Format

```
error[VSP-E406]: method `length` not found on type `int`
  --> src/main.vsp:10:8
   |
10 |     print(x.length())
   |            ^^^^^^^^
   |
   = help: did you mean `len(x)`?
   = note: `i32` does not have methods
```

Fields:

1. Severity (error, warning, note, help) and code in brackets.
2. Message.
3. Location: --> file:line:column.
4. Snippet with ^ underline.
5. Optional help and note lines.

18.2 JSON Format

With --json, diagnostics are emitted as JSON Lines, one per line:

```json
{"severity":"error","code":"VSP-E406","message":"method `length` not found on type `int`","span":{"file":"src/main.vsp","start":{"line":10,"column":8,"byte":245},"end":{"line":10,"column":16,"byte":253}},"notes":[{"kind":"help","message":"did you mean `len(x)`?"}]}
```

18.3 Ordering

Diagnostics are sorted by:

1. File path (lexicographic).
2. Start byte offset.
3. Severity (error before warning).
4. Code (lexicographic).

18.4 Warnings as Errors

--deny <lint> promotes a warning to an error. --deny warnings promotes all warnings. --forbid <lint> makes it impossible to silence (used in CI).

18.5 Diagnostic Limits

By default, the compiler emits at most 200 diagnostics per file. --max-errors <n> overrides. --max-errors 0 disables the limit.

18.6 Source Snippets

Snippets show 2 lines of context before and after the error location. Long lines are truncated at 100 characters with ….

18.7 Colors

Colors are used when --color always or stdout is a TTY. The palette:

· error — red
· warning — yellow
· note — cyan
· help — green
· code — bold

---

19. Exit Codes

Code Meaning
0 Success
1 Compilation error (source-level)
2 Usage error (bad flags, missing file)
3 I/O error
4 Internal compiler error
5 Missing dependency
6 Linker error
7 Signing error
101 Panic (compiler crash)

Exit codes are stable and may be relied upon by scripts.

---

20. Configuration Files

20.1 Discovery

vsc.toml is discovered by walking up from the current directory until the filesystem root. The first one found wins. --config <file> overrides.

20.2 Schema

```toml
[compiler]
opt_level = 1
debug = false
strip = false
incremental = true
target = "arc-vm-amd64"
jobs = 0        # 0 = auto

[compiler.warnings]
unused = "warn"       # or "allow", "deny"
deprecated = "warn"
shadowing = "allow"

[compiler.lints]
# Individual lint codes
"VSP-W006" = "deny"

[arc]
as_path = "/usr/local/bin/arc-as"
link_path = "/usr/local/bin/arc-link"
sign_path = "/usr/local/bin/arc-sign"
signing_key = "/home/user/.arc/signing.pem"

[output]
dir = "build"
emit = ["arc"]

[paths]
src = ["src"]
tests = ["tests"]
benches = ["benches"]
```

20.3 Precedence

Flags > environment > config file > defaults.

20.4 Project vs User Config

· vsc.toml in the project root is the project config.
· ~/.config/vsc/config.toml is the user config.
· Project config overrides user config.

---

21. Embedding as a Library

21.1 Go API

vsc is implemented in Go and exposes a public API for tooling:

```go
package vsc

type Options struct {
    OptLevel    int
    Debug       bool
    Target      string
    Warnings    map[string]Severity
    Config      *Config
}

type Result struct {
    Assembly []byte
    HIR      []byte
    LIR      []byte
    AST      []byte
    Diags    []Diagnostic
}

func Compile(source []byte, opts Options) (*Result, error)
func CompileFile(path string, opts Options) (*Result, error)
func CompileProject(root string, opts Options) (*Result, error)
```

21.2 AST Access

```go
type AST struct { ... }
func Parse(source []byte) (*AST, []Diagnostic)
func (a *AST) Walk(fn func(Node))
func (a *AST) Find(pos Position) Node
func (a *AST) Dump(w io.Writer)
```

21.3 Type Information

```go
type TypeInfo struct { ... }
func Check(ast *AST, opts Options) (*TypeInfo, []Diagnostic)
func (t *TypeInfo) TypeOf(node Node) Type
```

21.4 Thread Safety

All public functions are safe for concurrent use by multiple goroutines, provided each call passes its own input. Internal caches are synchronized.

21.5 Versioning

The API follows SemVer. Breaking changes require a major version bump. The API is stable within a major version.

---

22. Performance Requirements

22.1 Targets

Metric Target
Lexing 100 MB/s
Parsing 50 MB/s
Type checking 30 MB/s
HIR → LIR 40 MB/s
LIR → ASM 20 MB/s
Full compile (10 kloc project) < 500 ms
Incremental compile (1-line change) < 50 ms

22.2 Memory

· Peak memory for a 10 kloc project: < 200 MB.
· Cache memory for a 100 kloc project: < 2 GB.

22.3 Parallelism

· Module-level parallelism: modules are compiled in parallel.
· Within a module, stages are sequential.
· The number of parallel workers is --jobs or CPU count.

22.4 Profiling

vsc supports profiling flags:

· --cpuprofile <file> — write a CPU profile.
· --memprofile <file> — write a memory profile.
· --trace <file> — write an execution trace.

22.5 Benchmarks

The compiler repository includes benchmarks under bench/. CI runs them on every commit and reports regressions.

---

23. Cross-Compilation

23.1 Target Triples

Triple Meaning
arc-vm-amd64 ARC VM on x86-64 host
arc-vm-arm64 ARC VM on ARM64 host
arc-vm-wasm ARC VM in WebAssembly (future)

23.2 Availability

vsc ships as a single binary per host platform. Cross-compilation requires:

· The target's arc-as and arc-link.
· The target's standard library (bundled or downloaded).

23.3 Flags

```
vsc build --target arc-vm-arm64
```

23.4 Environment

VSC_TARGET sets the default target. VSC_SYSROOT points to a target-specific sysroot for standard library lookup.

---

24. Testing the Compiler

24.1 Test Categories

· Unit tests — one per compiler module.
· Snapshot tests — golden files for AST, HIR, LIR, ASM, and diagnostics.
· Fuzz tests — random input to the lexer and parser.
· Differential tests — compare optimized vs unoptimized output on a corpus.
· Conformance tests — a suite of programs whose behavior is fixed by the language spec.

24.2 Snapshot Tests

Every source file in tests/snapshots/ has expected outputs in tests/snapshots/expected/<name>.{ast,hir,lir,asm,diag}. Running vsc test-snapshots compares.

Updating snapshots: vsc test-snapshots --update.

24.3 Differential Tests

For each test program:

1. Compile at -O0 and run.
2. Compile at -O3 and run.
3. Compare outputs.

Any difference is a miscompilation.

24.4 Fuzz Tests

Lexer and parser fuzz tests use go-fuzz-style coverage-guided fuzzing. Findings are minimized and added to the corpus.

24.5 Conformance Suite

The conformance suite lives in tests/conformance/ and is versioned with the language specification. Each test contains:

· Input source.
· Expected diagnostics (or expected behavior).
· Reference to the spec section.

Every implementation claiming conformance must pass this suite.

24.6 Continuous Integration

CI runs:

· Unit tests.
· Snapshot tests.
· Conformance tests on every PR.
· Differential tests nightly.
· Fuzz tests for a bounded time on every PR.

---

Appendix A: Command Reference

```
vsc build [flags] [files...]
    Compile Vesper source to ARC.

vsc run [flags] [files...] [-- args...]
    Compile and run on ARC VM.

vsc check [flags] [files...]
    Parse and type-check without emitting output.

vsc test [flags] [files...]
    Compile and run tests.

vsc bench [flags] [files...]
    Compile and run benchmarks.

vsc clean
    Remove build artifacts and cache.

vsc version
    Print version and exit.

vsc help [command]
    Print help for a command.
```

---

Appendix B: Diagnostic Codes

Lexer

Code Meaning
VSP-E100 Invalid character
VSP-E101 Unterminated string literal
VSP-E102 Unterminated block comment
VSP-E103 Invalid escape sequence
VSP-E104 Invalid numeric literal
VSP-E110 Tab in indentation
VSP-E111 Mixed tabs and spaces

Parser

Code Meaning
VSP-E200 Expected token
VSP-E201 Unexpected end of file
VSP-E202 Unexpected indentation
VSP-E203 Expected colon
VSP-E204 Expected block
VSP-E205 Invalid pattern
VSP-E206 Invalid type expression
VSP-E207 Reserved keyword used as identifier

Resolver

Code Meaning
VSP-E300 Undefined variable
VSP-E301 Undefined function
VSP-E302 Undefined type
VSP-E303 Undefined module
VSP-E304 Symbol not exported
VSP-E305 Cyclic import
VSP-E306 Module path mismatch
VSP-E307 Duplicate definition
VSP-W003 Shadowing outer binding
VSP-W004 Shadowing prelude item

Type Checker

Code Meaning
VSP-E400 Cannot infer type
VSP-E401 Type mismatch
VSP-E402 Not a function
VSP-E403 Wrong number of arguments
VSP-E404 Missing named argument
VSP-E405 Unknown named argument
VSP-E406 Method not found
VSP-E407 Field not found
VSP-E408 Interface not satisfied
VSP-E409 Immutable binding reassigned
VSP-E410 Cannot move
VSP-E411 Use of moved value
VSP-E412 Non-exhaustive match
VSP-E413 Unreachable pattern
VSP-E414 ? in wrong context
VSP-E415 Generic bound not satisfied
VSP-E416 Cyclic type
VSP-W005 Lossy numeric conversion
VSP-W006 Unused variable
VSP-W007 Unused import
VSP-W008 Unused function
VSP-W009 Deprecated API
VSP-W010 Unreachable code

Codegen

Code Meaning
VSP-E500 Register allocation failure
VSP-E501 Unsupported operation
VSP-E502 Internal codegen error

---

Appendix C: IR Formats

C.1 AST Dump Format

```
Module(<name>)
├── Import(<path>)
├── Struct(<name>) [<span>]
│   ├── Field(<name>: <type>) [<span>]
│   └── ...
├── Method(<name>) [<span>]
│   ├── Receiver(<name>: <type>)
│   ├── Param(<name>: <type>)
│   └── Body
│       └── <expr tree>
└── Function(<name>) [<span>]
    └── Body
        └── <expr tree>
```

C.2 HIR Dump Format

```
fn <name>(<param>: <type>, ...) -> <type>:
  block:
    t0 = <op>(<args>)
    t1 = <op>(<args>)
    ret t1
```

C.3 LIR Dump Format

```
fn <name>:
  entry:
    %0: <type> = param 0
    %1: <type> = <op>(%0)
    br_cond %1, .then, .else

  then:
    %2: <type> = <op>()
    br .merge

  else:
    %3: <type> = <op>()
    br .merge

  merge:
    %4: <type> = phi(%2 from .then, %3 from .else)
    ret %4
```

C.4 Symbol Table Format

```
<kind> <mangled> <original> <type> <file>:<line>:<col>
```

Example:

```
fn     add          add          (i32,i32)->i32   src/math.vsp:3:1
method Point_length Point.length (Point)->f64     src/geometry/point.vsp:10:1
type   Point        Point        struct{...}       src/geometry/point.vsp:5:1
```

---

End of specification.
