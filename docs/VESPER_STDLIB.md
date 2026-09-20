The Vesper Standard Library Specification

Version 0.1 (Draft)
Package: std
Target: ARC Virtual Machine

---

Table of Contents

1. Introduction
2. Design Principles
3. Module Layout
4. Core Types
5. std.io — Input and Output
6. std.text — Text Processing
7. std.collections — Data Structures
8. std.math — Mathematics
9. std.time — Time and Clocks
10. std.fs — File System
11. std.path — Path Manipulation
12. std.process — Processes
13. std.os — Operating System
14. std.env — Environment
15. std.sync — Synchronization
16. std.crypto — Cryptography
17. std.rand — Random Numbers
18. std.json — JSON
19. std.regex — Regular Expressions
20. std.net — Networking
21. std.log — Logging
22. std.test — Testing
23. std.bench — Benchmarking
24. std.iter — Iterators
25. std.arena — Arena Allocation
26. std.intern — String Interning
27. std.bits — Bit Manipulation
28. std.sort — Sorting
29. std.hash — Hashing
30. std.compress — Compression
31. std.encoding — Encodings
32. std.unicode — Unicode
33. std.uuid — UUID
34. std.error — Error Handling
35. std.ffi — Foreign Function Interface
36. std.debug — Debugging
37. std.prof — Profiling
38. std.trace — Tracing
39. Conventions and Guarantees
40. Versioning and Stability

---

1. Introduction

The Vesper Standard Library (std) is the collection of modules that every Vesper program can use without importing external dependencies. It provides the fundamental abstractions of the language: input/output, text, collections, math, time, concurrency, cryptography, networking, and testing.

The standard library is designed to be:

· Complete. Everything needed to write real applications is included.
· Consistent. Names, signatures, and behavior follow the same conventions across all modules.
· Safe. No module exposes unsafe primitives without clear documentation.
· Portable. Where the underlying ARC VM provides a syscall, the standard library uses it. Where it does not, the library provides a portable fallback.
· Explicit. No hidden I/O, no hidden allocations, no hidden state.
· Documented. Every public function has a doc comment with an example.

This document specifies the public API of every module. It is normative: any conforming implementation must provide these functions with the specified signatures and behavior.

---

2. Design Principles

2.1 Principles

1. Errors are values. Functions that can fail return Result<T, E> or Option<T>. Exceptions are reserved for programming errors.
2. Explicit I/O. Reading or writing external state requires an explicit function call. No implicit logging, no hidden network requests.
3. Zero-cost abstractions. Wrappers around primitives compile to the same code as the primitive.
4. Composability. Functions compose via pipes (|>) and iterators.
5. No global state. Except for a few well-documented cases (std.rand.default(), std.log.default_logger()).
6. Deterministic where possible. Non-determinism (time, randomness, scheduling) is explicit.

2.2 Naming conventions

· Modules: lowercase, dotted (std.collections.map).
· Types: PascalCase (HashMap, Result).
· Functions: snake_case (read_file, to_string).
· Constants: UPPER_SNAKE_CASE (MAX_SIZE, PI).
· Predicates: prefix is_, has_, can_ (is_empty, has_key, can_read).

2.3 Parameter order

· The "subject" of a function comes first: text.split(sep), list.map(f).
· Options and configuration come last.
· Callbacks are usually last.

2.4 Result types

Every module defines its own error enum:

```vesper
enum IOError:
    NotFound
    PermissionDenied
    AlreadyExists
    InvalidInput(message: string)
    Other(message: string)
```

Functions return Result<T, IOError>, Result<T, ParseError>, etc. The ? operator propagates them.

---

3. Module Layout

3.1 Module tree

```
std
├── io
│   ├── reader
│   ├── writer
│   ├── buffered
│   └── files
├── text
│   ├── string
│   ├── bytes
│   ├── format
│   ├── parse
│   └── unicode
├── collections
│   ├── vec
│   ├── map
│   ├── set
│   ├── btree
│   ├── deque
│   ├── heap
│   └── bitvec
├── math
│   ├── int
│   ├── float
│   ├── bigint
│   ├── rational
│   └── complex
├── time
├── fs
├── path
├── process
├── os
├── env
├── sync
├── crypto
├── rand
├── json
├── regex
├── net
│   ├── tcp
│   ├── udp
│   ├── dns
│   └── tls
├── log
├── test
├── bench
├── iter
├── arena
├── intern
├── bits
├── sort
├── hash
├── compress
├── encoding
├── unicode
├── uuid
├── error
├── ffi
├── debug
├── prof
└── trace
```

3.2 What is in std vs external packages

std includes only:

· Abstractions used by nearly every program.
· Wrappers around ARC VM syscalls.
· Algorithms that are hard to implement correctly.

External packages (via vspm) provide:

· Domain-specific libraries (web frameworks, databases, game engines).
· Alternative implementations (e.g., fastjson, simd-math).
· Anything that depends on a specific platform.

---

4. Core Types

These types are part of the prelude and are always available.

4.1 Option<T>

```vesper
enum Option<T>:
    Some(value: T)
    None
```

Methods:

```vesper
impl<T> Option<T>:
    fn is_some(self) -> bool
    fn is_none(self) -> bool
    fn unwrap(self) -> T                        # panics if None
    fn unwrap_or(self, default: T) -> T
    fn unwrap_or_else(self, f: fn() -> T) -> T
    fn map<U>(self, f: fn(T) -> U) -> Option<U>
    fn and_then<U>(self, f: fn(T) -> Option<U>) -> Option<U>
    fn or(self, other: Option<T>) -> Option<T>
    fn filter(self, pred: fn(T) -> bool) -> Option<T>
    fn ok_or<E>(self, err: E) -> Result<T, E>
```

4.2 Result<T, E>

```vesper
enum Result<T, E>:
    Ok(value: T)
    Err(error: E)
```

Methods:

```vesper
impl<T, E> Result<T, E>:
    fn is_ok(self) -> bool
    fn is_err(self) -> bool
    fn unwrap(self) -> T                        # panics if Err
    fn unwrap_or(self, default: T) -> T
    fn unwrap_or_else(self, f: fn(E) -> T) -> T
    fn expect(self, msg: string) -> T
    fn map<U>(self, f: fn(T) -> U) -> Result<U, E>
    fn map_err<F>(self, f: fn(E) -> F) -> Result<T, F>
    fn and_then<U>(self, f: fn(T) -> Result<U, E>) -> Result<U, E>
    fn or(self, other: Result<T, E>) -> Result<T, E>
    fn ok(self) -> Option<T>
    fn err(self) -> Option<E>
```

4.3 Atom

Named constant, defined in the prelude. Type Atom, values via literals (:ok, :error).

4.4 Any

Universal interface implemented by all types.

4.5 Comparable

```vesper
interface Comparable:
    fn compare(self, other: Self) -> Ordering

enum Ordering:
    Less
    Equal
    Greater
```

4.6 Display

```vesper
interface Display:
    fn to_string(self) -> string
```

4.7 Hash

```vesper
interface Hash:
    fn hash(self, hasher: Hasher) -> unit
```

4.8 Iterator<T>

```vesper
interface Iterator<T>:
    fn next(self: Iterator<T>) -> Option<T>
```

---

5. std.io — Input and Output

5.1 Overview

std.io provides the fundamental I/O abstractions: readers, writers, and streams.

5.2 Interfaces

```vesper
interface Reader:
    fn read(self: Reader, buf: [u8]) -> Result<u32, IOError>

interface Writer:
    fn write(self: Writer, buf: [u8]) -> Result<u32, IOError>
    fn flush(self: Writer) -> Result<unit, IOError>

interface Seekable:
    fn seek(self: Seekable, pos: u64, whence: SeekFrom) -> Result<u64, IOError>

enum SeekFrom:
    Start
    Current
    End
```

5.3 Standard streams

```vesper
pub fn stdin() -> Reader
pub fn stdout() -> Writer
pub fn stderr() -> Writer
```

These return the process's standard streams. Repeated calls return the same object.

5.4 Reading

```vesper
pub fn read_all(r: Reader) -> Result<[u8], IOError>
pub fn read_line(r: Reader) -> Result<Option<string>, IOError>
pub fn read_to_string(r: Reader) -> Result<string, IOError>
pub fn read_exact(r: Reader, buf: [u8]) -> Result<unit, IOError>
```

read_line strips the trailing \n and returns None at EOF.

5.5 Writing

```vesper
pub fn write_all(w: Writer, buf: [u8]) -> Result<unit, IOError>
pub fn write_str(w: Writer, s: string) -> Result<unit, IOError>
pub fn write_line(w: Writer, s: string) -> Result<unit, IOError>
pub fn print(s: string)
pub fn println(s: string)
pub fn eprint(s: string)
pub fn eprintln(s: string)
```

print and println write to stdout; eprint* write to stderr.

5.6 Buffered I/O

```vesper
pub struct BufReader<R: Reader>:
    # internal

impl<R: Reader> BufReader<R>:
    pub fn new(r: R) -> BufReader<R>
    pub fn with_capacity(r: R, cap: u32) -> BufReader<R>

pub struct BufWriter<W: Writer>:
    # internal

impl<W: Writer> BufWriter<W>:
    pub fn new(w: W) -> BufWriter<W>
    pub fn with_capacity(w: W, cap: u32) -> BufWriter<W>
```

5.7 Files

See std.fs for file operations.

5.8 Pipes

```vesper
pub fn pipe() -> Result<(Reader, Writer), IOError>
```

Returns a pair of connected streams.

5.9 Error type

```vesper
enum IOError:
    NotFound
    PermissionDenied
    AlreadyExists
    InvalidInput(message: string)
    UnexpectedEof
    Interrupted
    BrokenPipe
    ConnectionRefused
    ConnectionReset
    Other(message: string)
```

---

6. std.text — Text Processing

6.1 Overview

std.text handles UTF-8 strings and byte sequences.

6.2 String basics

Strings are immutable UTF-8 sequences of bytes. The following functions are in the prelude or std.text:

```vesper
pub fn len(s: string) -> u32                    # bytes, not chars
pub fn is_empty(s: string) -> bool
pub fn char_count(s: string) -> u32             # Unicode code points
pub fn byte_at(s: string, i: u32) -> u8
pub fn char_at(s: string, i: u32) -> Option<char>
pub fn substring(s: string, lo: u32, hi: u32) -> string
pub fn slice_bytes(s: string, lo: u32, hi: u32) -> string
```

6.3 Searching

```vesper
pub fn contains(s: string, sub: string) -> bool
pub fn starts_with(s: string, prefix: string) -> bool
pub fn ends_with(s: string, suffix: string) -> bool
pub fn find(s: string, sub: string) -> Option<u32>
pub fn rfind(s: string, sub: string) -> Option<u32>
pub fn find_char(s: string, c: char) -> Option<u32>
```

6.4 Splitting and joining

```vesper
pub fn split(s: string, sep: string) -> [string]
pub fn split_char(s: string, c: char) -> [string]
pub fn split_whitespace(s: string) -> [string]
pub fn split_lines(s: string) -> [string]
pub fn lines(s: string) -> Iterator<string>
pub fn join(parts: [string], sep: string) -> string
```

6.5 Transformations

```vesper
pub fn trim(s: string) -> string
pub fn trim_start(s: string) -> string
pub fn trim_end(s: string) -> string
pub fn trim_matches(s: string, chars: string) -> string
pub fn to_lowercase(s: string) -> string
pub fn to_uppercase(s: string) -> string
pub fn replace(s: string, from: string, to: string) -> string
pub fn replace_all(s: string, from: string, to: string) -> string
pub fn repeat(s: string, n: u32) -> string
pub fn reverse(s: string) -> string             # reverses code points
```

6.6 Padding

```vesper
pub fn pad_start(s: string, width: u32, c: char) -> string
pub fn pad_end(s: string, width: u32, c: char) -> string
pub fn center(s: string, width: u32, c: char) -> string
```

6.7 Parsing

```vesper
pub fn parse_int(s: string) -> Result<i32, ParseError>
pub fn parse_int_radix(s: string, radix: u32) -> Result<i32, ParseError>
pub fn parse_i64(s: string) -> Result<i64, ParseError>
pub fn parse_u64(s: string) -> Result<u64, ParseError>
pub fn parse_f64(s: string) -> Result<f64, ParseError>
pub fn parse_bool(s: string) -> Result<bool, ParseError>
```

6.8 Formatting

```vesper
pub fn to_string<T: Display>(x: T) -> string
pub fn format(fmt: string, args: [Any]) -> string
pub fn hex(n: u32) -> string
pub fn hex_u64(n: u64) -> string
pub fn bin(n: u32) -> string
pub fn oct(n: u32) -> string
```

Format string syntax:

```
{}          default display
{:d}        integer
{:x}        hexadecimal
{:o}        octal
{:b}        binary
{:#x}       hex with 0x prefix
{:08x}      hex, zero-padded to 8 chars
{:>10}      right-aligned in 10 chars
{:<10}      left-aligned
{:^10}      centered
{:.3}       precision (floats)
{:e}        scientific notation
```

6.9 Bytes

```vesper
pub struct Bytes:
    # immutable byte sequence

impl Bytes:
    pub fn from_string(s: string) -> Bytes
    pub fn from_array(b: [u8]) -> Bytes
    pub fn to_string(self) -> string            # UTF-8 lossy
    pub fn len(self) -> u32
    pub fn get(self, i: u32) -> Option<u8>
    pub fn slice(self, lo: u32, hi: u32) -> Bytes
```

6.10 Error type

```vesper
enum ParseError:
    Empty
    InvalidCharacter(c: char, position: u32)
    Overflow
    Underflow
    InvalidFormat(expected: string)
```

---

7. std.collections — Data Structures

7.1 Overview

Collections are generic, safe, and optimized for common cases.

7.2 Vec<T> — dynamic array

```vesper
pub struct Vec<T>:
    # internal

impl<T> Vec<T>:
    pub fn new() -> Vec<T>
    pub fn with_capacity(cap: u32) -> Vec<T>
    pub fn len(self) -> u32
    pub fn capacity(self) -> u32
    pub fn is_empty(self) -> bool
    pub fn push(self: Vec<T>, value: T)
    pub fn pop(self: Vec<T>) -> Option<T>
    pub fn get(self, i: u32) -> Option<T>
    pub fn set(self: Vec<T>, i: u32, value: T) -> Result<unit, IndexError>
    pub fn insert(self: Vec<T>, i: u32, value: T) -> Result<unit, IndexError>
    pub fn remove(self: Vec<T>, i: u32) -> Result<T, IndexError>
    pub fn clear(self: Vec<T>)
    pub fn truncate(self: Vec<T>, len: u32)
    pub fn reserve(self: Vec<T>, additional: u32)
    pub fn shrink_to_fit(self: Vec<T>)
    pub fn iter(self) -> Iterator<T>
    pub fn map<U>(self, f: fn(T) -> U) -> Vec<U>
    pub fn filter(self, pred: fn(T) -> bool) -> Vec<T>
    pub fn fold<A>(self, init: A, f: fn(A, T) -> A) -> A
    pub fn sort(self: Vec<T>) where T: Comparable
    pub fn sort_by(self: Vec<T>, cmp: fn(T, T) -> Ordering)
    pub fn reverse(self: Vec<T>)
    pub fn contains(self, value: T) -> bool where T: Comparable
```

Vec<T> implements Iterator<T>, Display, Comparable (when T is comparable), and Hash.

7.3 HashMap<K, V> — hash table

```vesper
pub struct HashMap<K: Hash + Comparable, V>:
    # internal

impl<K: Hash + Comparable, V> HashMap<K, V>:
    pub fn new() -> HashMap<K, V>
    pub fn with_capacity(cap: u32) -> HashMap<K, V>
    pub fn len(self) -> u32
    pub fn is_empty(self) -> bool
    pub fn insert(self: HashMap<K, V>, key: K, value: V) -> Option<V>
    pub fn get(self, key: K) -> Option<V>
    pub fn get_or_insert(self: HashMap<K, V>, key: K, default: V) -> V
    pub fn remove(self: HashMap<K, V>, key: K) -> Option<V>
    pub fn contains_key(self, key: K) -> bool
    pub fn clear(self: HashMap<K, V>)
    pub fn keys(self) -> Iterator<K>
    pub fn values(self) -> Iterator<V>
    pub fn iter(self) -> Iterator<(K, V)>
    pub fn entry(self: HashMap<K, V>, key: K) -> Entry<K, V>

enum Entry<K, V>:
    Occupied(value: V)
    Vacant(key: K)
```

7.4 HashSet<T>

```vesper
pub struct HashSet<T: Hash + Comparable>:
    # internal

impl<T: Hash + Comparable> HashSet<T>:
    pub fn new() -> HashSet<T>
    pub fn insert(self: HashSet<T>, value: T) -> bool
    pub fn remove(self: HashSet<T>, value: T) -> bool
    pub fn contains(self, value: T) -> bool
    pub fn len(self) -> u32
    pub fn iter(self) -> Iterator<T>
    pub fn union(self, other: HashSet<T>) -> HashSet<T>
    pub fn intersection(self, other: HashSet<T>) -> HashSet<T>
    pub fn difference(self, other: HashSet<T>) -> HashSet<T>
```

7.5 BTreeMap<K, V> — sorted map

```vesper
pub struct BTreeMap<K: Comparable, V>:
    # internal

impl<K: Comparable, V> BTreeMap<K, V>:
    pub fn new() -> BTreeMap<K, V>
    pub fn insert(self: BTreeMap<K, V>, key: K, value: V) -> Option<V>
    pub fn get(self, key: K) -> Option<V>
    pub fn remove(self: BTreeMap<K, V>, key: K) -> Option<V>
    pub fn len(self) -> u32
    pub fn first_key(self) -> Option<K>
    pub fn last_key(self) -> Option<K>
    pub fn range(self, lo: K, hi: K) -> Iterator<(K, V)>
    pub fn iter(self) -> Iterator<(K, V)>
```

7.6 Deque<T> — double-ended queue

```vesper
pub struct Deque<T>:
    # internal

impl<T> Deque<T>:
    pub fn new() -> Deque<T>
    pub fn push_front(self: Deque<T>, value: T)
    pub fn push_back(self: Deque<T>, value: T)
    pub fn pop_front(self: Deque<T>) -> Option<T>
    pub fn pop_back(self: Deque<T>) -> Option<T>
    pub fn front(self) -> Option<T>
    pub fn back(self) -> Option<T>
    pub fn len(self) -> u32
    pub fn iter(self) -> Iterator<T>
```

7.7 BinaryHeap<T> — priority queue

```vesper
pub struct BinaryHeap<T: Comparable>:
    # internal

impl<T: Comparable> BinaryHeap<T>:
    pub fn new() -> BinaryHeap<T>       # max-heap
    pub fn push(self: BinaryHeap<T>, value: T)
    pub fn pop(self: BinaryHeap<T>) -> Option<T>
    pub fn peek(self) -> Option<T>
    pub fn len(self) -> u32
```

7.8 BitVec — bit vector

```vesper
pub struct BitVec:
    # internal

impl BitVec:
    pub fn new() -> BitVec
    pub fn with_capacity(bits: u32) -> BitVec
    pub fn len(self) -> u32
    pub fn get(self, i: u32) -> bool
    pub fn set(self: BitVec, i: u32, value: bool)
    pub fn push(self: BitVec, value: bool)
    pub fn pop(self: BitVec) -> Option<bool>
    pub fn count_ones(self) -> u32
    pub fn count_zeros(self) -> u32
```

7.9 Error type

```vesper
enum IndexError:
    OutOfBounds(index: u32, len: u32)
```

---

8. std.math — Mathematics

8.1 Integer math

```vesper
pub const I32_MIN: i32
pub const I32_MAX: i32
pub const U32_MAX: u32
# ... similar for other widths

pub fn abs(x: i32) -> i32
pub fn min(a: i32, b: i32) -> i32
pub fn max(a: i32, b: i32) -> i32
pub fn clamp(x: i32, lo: i32, hi: i32) -> i32
pub fn gcd(a: u32, b: u32) -> u32
pub fn lcm(a: u32, b: u32) -> u32
pub fn pow(base: i64, exp: u32) -> i64              # overflow: Result
pub fn checked_add(a: i32, b: i32) -> Option<i32>
pub fn checked_sub(a: i32, b: i32) -> Option<i32>
pub fn checked_mul(a: i32, b: i32) -> Option<i32>
pub fn checked_div(a: i32, b: i32) -> Option<i32>
pub fn wrapping_add(a: i32, b: i32) -> i32
pub fn saturating_add(a: i32, b: i32) -> i32
```

8.2 Float math

```vesper
pub const PI: f64 = 3.141592653589793
pub const E: f64  = 2.718281828459045
pub const INF: f64
pub const NAN: f64

pub fn abs(x: f64) -> f64
pub fn sqrt(x: f64) -> f64
pub fn cbrt(x: f64) -> f64
pub fn pow(x: f64, y: f64) -> f64
pub fn exp(x: f64) -> f64
pub fn ln(x: f64) -> f64
pub fn log2(x: f64) -> f64
pub fn log10(x: f64) -> f64
pub fn sin(x: f64) -> f64
pub fn cos(x: f64) -> f64
pub fn tan(x: f64) -> f64
pub fn asin(x: f64) -> f64
pub fn acos(x: f64) -> f64
pub fn atan(x: f64) -> f64
pub fn atan2(y: f64, x: f64) -> f64
pub fn sinh(x: f64) -> f64
pub fn cosh(x: f64) -> f64
pub fn tanh(x: f64) -> f64
pub fn floor(x: f64) -> f64
pub fn ceil(x: f64) -> f64
pub fn round(x: f64) -> f64
pub fn trunc(x: f64) -> f64
pub fn fract(x: f64) -> f64
pub fn is_nan(x: f64) -> bool
pub fn is_infinite(x: f64) -> bool
pub fn is_finite(x: f64) -> bool
pub fn sign(x: f64) -> f64
pub fn hypot(x: f64, y: f64) -> f64
pub fn to_degrees(x: f64) -> f64
pub fn to_radians(x: f64) -> f64
```

8.3 BigInt

Arbitrary-precision integers.

```vesper
pub struct BigInt:
    # internal

impl BigInt:
    pub fn zero() -> BigInt
    pub fn one() -> BigInt
    pub fn from_i64(x: i64) -> BigInt
    pub fn from_string(s: string) -> Result<BigInt, ParseError>
    pub fn to_string(self) -> string
    pub fn to_i64(self) -> Option<i64>
    pub fn add(self, other: BigInt) -> BigInt
    pub fn sub(self, other: BigInt) -> BigInt
    pub fn mul(self, other: BigInt) -> BigInt
    pub fn div(self, other: BigInt) -> BigInt
    pub fn rem(self, other: BigInt) -> BigInt
    pub fn pow(self, exp: u32) -> BigInt
    pub fn neg(self) -> BigInt
    pub fn abs(self) -> BigInt
    pub fn compare(self, other: BigInt) -> Ordering
    pub fn is_zero(self) -> bool
    pub fn is_negative(self) -> bool
```

8.4 Rational

Exact fractions.

```vesper
pub struct Rational:
    # internal

impl Rational:
    pub fn new(num: BigInt, den: BigInt) -> Rational
    pub fn from_i64(num: i64, den: i64) -> Rational
    pub fn num(self) -> BigInt
    pub fn den(self) -> BigInt
    pub fn add(self, other: Rational) -> Rational
    pub fn sub(self, other: Rational) -> Rational
    pub fn mul(self, other: Rational) -> Rational
    pub fn div(self, other: Rational) -> Rational
    pub fn to_f64(self) -> f64
```

8.5 Complex

```vesper
pub struct Complex:
    re: f64
    im: f64

impl Complex:
    pub fn new(re: f64, im: f64) -> Complex
    pub fn add(self, other: Complex) -> Complex
    pub fn sub(self, other: Complex) -> Complex
    pub fn mul(self, other: Complex) -> Complex
    pub fn div(self, other: Complex) -> Complex
    pub fn abs(self) -> f64
    pub fn arg(self) -> f64
    pub fn conj(self) -> Complex
```

---

9. std.time — Time and Clocks

9.1 Types

```vesper
pub struct Instant:
    # monotonic time, opaque

pub struct Duration:
    seconds: i64
    nanos: i32

pub struct DateTime:
    year: i32
    month: u8       # 1..12
    day: u8         # 1..31
    hour: u8        # 0..23
    minute: u8      # 0..59
    second: u8      # 0..59
    nanos: u32
    offset: i32     # seconds from UTC
```

9.2 Clocks

```vesper
pub fn now() -> Instant
pub fn now_utc() -> DateTime
pub fn now_local() -> DateTime
pub fn elapsed(start: Instant) -> Duration
pub fn since(start: Instant) -> Duration
```

9.3 Duration

```vesper
impl Duration:
    pub fn from_seconds(s: i64) -> Duration
    pub fn from_millis(ms: i64) -> Duration
    pub fn from_micros(us: i64) -> Duration
    pub fn from_nanos(ns: i64) -> Duration
    pub fn seconds(self) -> i64
    pub fn millis(self) -> i64
    pub fn micros(self) -> i64
    pub fn nanos(self) -> i64
    pub fn add(self, other: Duration) -> Duration
    pub fn sub(self, other: Duration) -> Duration
    pub fn mul(self, n: i64) -> Duration
    pub fn is_zero(self) -> bool
    pub fn is_negative(self) -> bool
```

9.4 DateTime operations

```vesper
impl DateTime:
    pub fn format(self, fmt: string) -> string
    pub fn parse(s: string, fmt: string) -> Result<DateTime, ParseError>
    pub fn to_utc(self) -> DateTime
    pub fn to_local(self) -> DateTime
    pub fn to_unix(self) -> i64
    pub fn from_unix(secs: i64) -> DateTime
    pub fn add_duration(self, d: Duration) -> DateTime
    pub fn sub(self, other: DateTime) -> Duration
    pub fn weekday(self) -> Weekday
    pub fn is_leap_year(self) -> bool
```

Format specifiers: %Y %m %d %H %M %S %f %z %Z %A %a %B %b.

9.5 Sleep

```vesper
pub fn sleep(d: Duration)
pub fn sleep_millis(ms: u32)
pub fn sleep_seconds(s: u32)
```

9.6 Timers

```vesper
pub struct Timer:
    # internal

impl Timer:
    pub fn start() -> Timer
    pub fn elapsed(self) -> Duration
    pub fn reset(self: Timer)
```

---

10. std.fs — File System

10.1 Types

```vesper
pub struct File:
    # internal

pub struct Metadata:
    pub size: u64
    pub modified: DateTime
    pub created: DateTime
    pub is_dir: bool
    pub is_file: bool
    pub is_symlink: bool
    pub permissions: Permissions

pub struct Permissions:
    pub readonly: bool
```

10.2 Opening and creating

```vesper
pub fn open(path: string) -> Result<File, IOError>
pub fn open_read(path: string) -> Result<File, IOError>
pub fn open_write(path: string) -> Result<File, IOError>
pub fn open_append(path: string) -> Result<File, IOError>
pub fn create(path: string) -> Result<File, IOError>
pub fn create_dir(path: string) -> Result<unit, IOError>
pub fn create_dir_all(path: string) -> Result<unit, IOError>
```

10.3 Reading and writing whole files

```vesper
pub fn read(path: string) -> Result<[u8], IOError>
pub fn read_string(path: string) -> Result<string, IOError>
pub fn write(path: string, contents: [u8]) -> Result<unit, IOError>
pub fn write_string(path: string, contents: string) -> Result<unit, IOError>
pub fn append_string(path: string, contents: string) -> Result<unit, IOError>
```

10.4 File operations

```vesper
impl File:
    pub fn read(self: File, buf: [u8]) -> Result<u32, IOError>
    pub fn write(self: File, buf: [u8]) -> Result<u32, IOError>
    pub fn seek(self: File, pos: u64, whence: SeekFrom) -> Result<u64, IOError>
    pub fn flush(self: File) -> Result<unit, IOError>
    pub fn metadata(self) -> Result<Metadata, IOError>
    pub fn set_len(self: File, len: u64) -> Result<unit, IOError>
    pub fn sync_all(self: File) -> Result<unit, IOError>

impl File: Reader
impl File: Writer
impl File: Seekable
```

10.5 Directory operations

```vesper
pub fn read_dir(path: string) -> Result<[DirEntry], IOError>

pub struct DirEntry:
    pub name: string
    pub path: string
    pub is_dir: bool
    pub is_file: bool
    pub is_symlink: bool

pub fn remove_file(path: string) -> Result<unit, IOError>
pub fn remove_dir(path: string) -> Result<unit, IOError>
pub fn remove_dir_all(path: string) -> Result<unit, IOError>
pub fn rename(from: string, to: string) -> Result<unit, IOError>
pub fn copy(from: string, to: string) -> Result<u64, IOError>
pub fn hard_link(from: string, to: string) -> Result<unit, IOError>
pub fn symlink(target: string, link: string) -> Result<unit, IOError>
pub fn read_link(path: string) -> Result<string, IOError>
```

10.6 Metadata

```vesper
pub fn metadata(path: string) -> Result<Metadata, IOError>
pub fn symlink_metadata(path: string) -> Result<Metadata, IOError>
pub fn exists(path: string) -> bool
pub fn is_file(path: string) -> bool
pub fn is_dir(path: string) -> bool
```

10.7 Walking

```vesper
pub fn walk(path: string) -> Iterator<Result<DirEntry, IOError>>
pub fn walk_recursive(path: string) -> Iterator<Result<DirEntry, IOError>>
pub fn glob(pattern: string) -> Result<[string], GlobError>
```

10.8 Temporary files

```vesper
pub fn temp_file() -> Result<File, IOError>
pub fn temp_dir() -> string
pub fn create_temp_dir(prefix: string) -> Result<string, IOError>
```

---

11. std.path — Path Manipulation

11.1 Types

```vesper
pub struct Path:
    # internal

pub struct PathBuf:
    # internal, mutable
```

11.2 Operations

```vesper
impl Path:
    pub fn new(s: string) -> Path
    pub fn as_str(self) -> string
    pub fn join(self, other: string) -> PathBuf
    pub fn parent(self) -> Option<Path>
    pub fn file_name(self) -> Option<string>
    pub fn extension(self) -> Option<string>
    pub fn stem(self) -> Option<string>
    pub fn is_absolute(self) -> bool
    pub fn is_relative(self) -> bool
    pub fn components(self) -> Iterator<string>
    pub fn to_absolute(self) -> Result<PathBuf, IOError>
    pub fn canonicalize(self) -> Result<PathBuf, IOError>
```

11.3 Free functions

```vesper
pub fn join(parts: [string]) -> string
pub fn normalize(path: string) -> string
pub fn is_absolute(path: string) -> bool
pub fn is_relative(path: string) -> bool
pub fn separator() -> char               # '/' on Unix, '\\' on Windows
pub fn current_dir() -> Result<string, IOError>
pub fn set_current_dir(path: string) -> Result<unit, IOError>
pub fn home_dir() -> Option<string>
pub fn config_dir() -> Option<string>
pub fn cache_dir() -> Option<string>
pub fn data_dir() -> Option<string>
pub fn temp_dir() -> string
```

---

12. std.process — Processes

12.1 Types

```vesper
pub struct Command:
    # internal

pub struct Child:
    pub id: u32
    # internal

pub struct Output:
    pub status: ExitStatus
    pub stdout: [u8]
    pub stderr: [u8]

pub struct ExitStatus:
    pub code: Option<i32>
    pub signal: Option<i32>

pub struct Stdio:
    Inherit
    Piped
    Null
```

12.2 Command

```vesper
impl Command:
    pub fn new(program: string) -> Command
    pub fn arg(self: Command, arg: string) -> Command
    pub fn args(self: Command, args: [string]) -> Command
    pub fn env(self: Command, key: string, value: string) -> Command
    pub fn env_clear(self: Command) -> Command
    pub fn current_dir(self: Command, dir: string) -> Command
    pub fn stdin(self: Command, stdio: Stdio) -> Command
    pub fn stdout(self: Command, stdio: Stdio) -> Command
    pub fn stderr(self: Command, stdio: Stdio) -> Command
    pub fn spawn(self) -> Result<Child, ProcessError>
    pub fn output(self) -> Result<Output, ProcessError>
    pub fn status(self) -> Result<ExitStatus, ProcessError>
```

12.3 Child

```vesper
impl Child:
    pub fn id(self) -> u32
    pub fn wait(self) -> Result<ExitStatus, ProcessError>
    pub fn try_wait(self) -> Result<Option<ExitStatus>, ProcessError>
    pub fn kill(self) -> Result<unit, ProcessError>
    pub fn stdin(self) -> Option<Writer>
    pub fn stdout(self) -> Option<Reader>
    pub fn stderr(self) -> Option<Reader>
```

12.4 Current process

```vesper
pub fn id() -> u32
pub fn exit(code: i32)
pub fn abort()
pub fn args() -> [string]
pub fn args_os() -> [string]
```

12.5 Error type

```vesper
enum ProcessError:
    NotFound(program: string)
    PermissionDenied
    SpawnFailed(message: string)
    WaitFailed(message: string)
```

---

13. std.os — Operating System

13.1 Constants

```vesper
pub const FAMILY: string          # "unix", "windows", "wasm"
pub const NAME: string            # "linux", "macos", "windows"
pub const ARCH: string            # "amd64", "arm64"
```

13.2 Operations

```vesper
pub fn exit(code: i32) -> !
pub fn abort() -> !
pub fn hostname() -> Result<string, OSError>
pub fn cpu_count() -> u32
pub fn physical_memory() -> Result<u64, OSError>
pub fn set_env(key: string, value: string)
pub fn remove_env(key: string)
pub fn get_env(key: string) -> Option<string>
pub fn env_vars() -> Iterator<(string, string)>
```

13.3 Signals (Unix)

```vesper
pub const SIGINT: i32
pub const SIGTERM: i32
pub const SIGKILL: i32
pub const SIGHUP: i32
# ...

pub fn raise(sig: i32) -> Result<unit, OSError>
pub fn handle_signal(sig: i32, handler: fn())
```

---

14. std.env — Environment

Deprecated in favor of std.os and std.process. Kept for compatibility.

```vesper
pub fn get(key: string) -> Option<string>
pub fn set(key: string, value: string)
pub fn remove(key: string)
pub fn vars() -> Iterator<(string, string)>
pub fn args() -> [string]
pub fn current_dir() -> Result<string, IOError>
```

---

15. std.sync — Synchronization

15.1 Mutex

```vesper
pub struct Mutex<T>:
    # internal

impl<T> Mutex<T>:
    pub fn new(value: T) -> Mutex<T>
    pub fn lock(self) -> MutexGuard<T>
    pub fn try_lock(self) -> Option<MutexGuard<T>>
    pub fn into_inner(self) -> T

pub struct MutexGuard<T>:
    # internal, dereferences to T

impl<T> MutexGuard<T>:
    # drop unlocks the mutex
```

15.2 RwLock

```vesper
pub struct RwLock<T>:
    # internal

impl<T> RwLock<T>:
    pub fn new(value: T) -> RwLock<T>
    pub fn read(self) -> RwLockReadGuard<T>
    pub fn write(self) -> RwLockWriteGuard<T>
    pub fn try_read(self) -> Option<RwLockReadGuard<T>>
    pub fn try_write(self) -> Option<RwLockWriteGuard<T>>
```

15.3 Atomic types

```vesper
pub struct AtomicBool:
    # internal

pub struct AtomicI32:
    # internal

pub struct AtomicI64:
    # internal

pub struct AtomicU32:
    # internal

pub struct AtomicU64:
    # internal

impl AtomicU32:
    pub fn new(value: u32) -> AtomicU32
    pub fn load(self, order: Ordering) -> u32
    pub fn store(self: AtomicU32, value: u32, order: Ordering)
    pub fn swap(self: AtomicU32, value: u32, order: Ordering) -> u32
    pub fn compare_exchange(self: AtomicU32, expected: u32, new: u32, success: Ordering, failure: Ordering) -> Result<u32, u32>
    pub fn fetch_add(self: AtomicU32, value: u32, order: Ordering) -> u32
    pub fn fetch_sub(self: AtomicU32, value: u32, order: Ordering) -> u32
    pub fn fetch_and(self: AtomicU32, value: u32, order: Ordering) -> u32
    pub fn fetch_or(self: AtomicU32, value: u32, order: Ordering) -> u32
    pub fn fetch_xor(self: AtomicU32, value: u32, order: Ordering) -> u32

enum Ordering:
    Relaxed
    Acquire
    Release
    AcqRel
    SeqCst
```

15.4 Once

```vesper
pub struct Once:
    # internal

impl Once:
    pub fn new() -> Once
    pub fn call_once(self: Once, f: fn())
    pub fn is_completed(self) -> bool
```

15.5 Condition variable

```vesper
pub struct Condvar:
    # internal

impl Condvar:
    pub fn new() -> Condvar
    pub fn wait(self: Condvar, guard: MutexGuard<T>) -> MutexGuard<T>
    pub fn wait_timeout(self: Condvar, guard: MutexGuard<T>, d: Duration) -> (MutexGuard<T>, bool)
    pub fn notify_one(self: Condvar)
    pub fn notify_all(self: Condvar)
```

15.6 Semaphore

```vesper
pub struct Semaphore:
    # internal

impl Semaphore:
    pub fn new(permits: u32) -> Semaphore
    pub fn acquire(self) -> SemaphoreGuard
    pub fn try_acquire(self) -> Option<SemaphoreGuard>
```

15.7 Barrier

```vesper
pub struct Barrier:
    # internal

impl Barrier:
    pub fn new(count: u32) -> Barrier
    pub fn wait(self) -> BarrierWaitResult
```

---

16. std.crypto — Cryptography

16.1 Overview

All cryptographic operations in this module are constant-time by default. Side-channel-resistant variants are the default, not an option.

16.2 Hash functions

```vesper
pub fn sha256(data: [u8]) -> [u8; 32]
pub fn sha512(data: [u8]) -> [u8; 64]
pub fn sha3_256(data: [u8]) -> [u8; 32]
pub fn sha3_512(data: [u8]) -> [u8; 64]
pub fn blake2b(data: [u8], out_len: u32) -> [u8]
pub fn blake3(data: [u8]) -> [u8; 32]

pub struct Sha256:
    # streaming interface

impl Sha256:
    pub fn new() -> Sha256
    pub fn update(self: Sha256, data: [u8])
    pub fn finalize(self) -> [u8; 32]
```

16.3 MAC

```vesper
pub fn hmac_sha256(key: [u8], data: [u8]) -> [u8; 32]
pub fn hmac_sha512(key: [u8], data: [u8]) -> [u8; 64]
pub fn poly1305(key: [u8; 32], data: [u8]) -> [u8; 16]
```

16.4 Symmetric ciphers

```vesper
pub struct Aes256Gcm:
    # internal

impl Aes256Gcm:
    pub fn new(key: [u8; 32]) -> Aes256Gcm
    pub fn encrypt(self, nonce: [u8; 12], plaintext: [u8], aad: [u8]) -> Result<[u8], CryptoError>
    pub fn decrypt(self, nonce: [u8; 12], ciphertext: [u8], aad: [u8]) -> Result<[u8], CryptoError>

pub struct ChaCha20Poly1305:
    # internal

impl ChaCha20Poly1305:
    pub fn new(key: [u8; 32]) -> ChaCha20Poly1305
    pub fn encrypt(self, nonce: [u8; 12], plaintext: [u8], aad: [u8]) -> [u8]
    pub fn decrypt(self, nonce: [u8; 12], ciphertext: [u8], aad: [u8]) -> Result<[u8], CryptoError>
```

16.5 Public-key cryptography

```vesper
pub struct Ed25519PrivateKey:
    # internal

pub struct Ed25519PublicKey:
    # internal

impl Ed25519PrivateKey:
    pub fn generate() -> Ed25519PrivateKey
    pub fn from_bytes(b: [u8; 32]) -> Ed25519PrivateKey
    pub fn to_bytes(self) -> [u8; 32]
    pub fn public_key(self) -> Ed25519PublicKey
    pub fn sign(self, message: [u8]) -> [u8; 64]

impl Ed25519PublicKey:
    pub fn from_bytes(b: [u8; 32]) -> Ed25519PublicKey
    pub fn to_bytes(self) -> [u8; 32]
    pub fn verify(self, message: [u8], signature: [u8; 64]) -> bool
```

Similar APIs exist for X25519, ECDSA, RSA.

16.6 Key derivation

```vesper
pub fn hkdf_sha256(ikm: [u8], salt: [u8], info: [u8], out_len: u32) -> [u8]
pub fn pbkdf2_sha256(password: [u8], salt: [u8], iterations: u32, out_len: u32) -> [u8]
pub fn argon2id(password: [u8], salt: [u8], config: Argon2Config) -> Result<[u8], CryptoError>

pub struct Argon2Config:
    pub memory_kib: u32
    pub iterations: u32
    pub parallelism: u32
    pub output_len: u32
```

16.7 Constant-time comparison

```vesper
pub fn ct_eq(a: [u8], b: [u8]) -> bool          # constant-time
pub fn ct_select(cond: bool, a: u32, b: u32) -> u32
pub fn ct_zeroize(buf: [u8])
```

16.8 Random

See std.rand.

16.9 Error type

```vesper
enum CryptoError:
    InvalidKeyLength
    InvalidNonce
    AuthenticationFailed
    InvalidSignature
    InsufficientEntropy
    Other(message: string)
```

---

17. std.rand — Random Numbers

17.1 Secure RNG

```vesper
pub struct SecureRng:
    # internal

impl SecureRng:
    pub fn new() -> Result<SecureRng, RandError>
    pub fn from_seed(seed: [u8; 32]) -> SecureRng
    pub fn fill_bytes(self: SecureRng, buf: [u8])
    pub fn next_u32(self: SecureRng) -> u32
    pub fn next_u64(self: SecureRng) -> u64
    pub fn next_f64(self: SecureRng) -> f64
    pub fn range_u32(self: SecureRng, lo: u32, hi: u32) -> u32
```

17.2 Fast non-cryptographic RNG

```vesper
pub struct Rng:
    # internal

impl Rng:
    pub fn new() -> Rng
    pub fn from_seed(seed: u64) -> Rng
    pub fn next_u32(self: Rng) -> u32
    pub fn next_u64(self: Rng) -> u64
    pub fn next_f64(self: Rng) -> f64
    pub fn range(self: Rng, lo: i32, hi: i32) -> i32
    pub fn bool(self: Rng) -> bool
    pub fn shuffle<T>(self: Rng, list: [T]) -> [T]
    pub fn choose<T>(self: Rng, list: [T]) -> Option<T>
```

17.3 Global convenience

```vesper
pub fn rand_int(lo: i32, hi: i32) -> i32
pub fn rand_float() -> f64
pub fn rand_bytes(n: u32) -> [u8]         # cryptographically secure
```

17.4 Error type

```vesper
enum RandError:
    NoEntropy
    InvalidSeed
```

---

18. std.json — JSON

18.1 Types

```vesper
pub enum Value:
    Null
    Bool(value: bool)
    Number(value: f64)
    Integer(value: i64)
    String(value: string)
    Array(items: [Value])
    Object(fields: {string: Value})
```

18.2 Parsing

```vesper
pub fn parse(s: string) -> Result<Value, JsonError>
pub fn parse_bytes(b: [u8]) -> Result<Value, JsonError]
pub struct Parser:
    # streaming parser

impl Parser:
    pub fn new(r: Reader) -> Parser
    pub fn next(self: Parser) -> Result<Option<Value>, JsonError>
```

18.3 Serialization

```vesper
pub fn stringify(value: Value) -> string
pub fn stringify_pretty(value: Value, indent: u32) -> string
pub fn write(w: Writer, value: Value) -> Result<unit, IOError>
```

18.4 Typed access

```vesper
pub fn get_bool(v: Value, path: string) -> Result<bool, JsonError>
pub fn get_int(v: Value, path: string) -> Result<i64, JsonError>
pub fn get_float(v: Value, path: string) -> Result<f64, JsonError>
pub fn get_string(v: Value, path: string) -> Result<string, JsonError]
pub fn get_array(v: Value, path: string) -> Result<[Value], JsonError]
pub fn get_object(v: Value, path: string) -> Result<{string: Value}, JsonError]
```

18.5 Marshaling

```vesper
pub fn marshal<T: Serializable>(value: T) -> string
pub fn unmarshal<T: Deserializable>(s: string) -> Result<T, JsonError]

interface Serializable:
    fn serialize(self) -> Value

interface Deserializable:
    fn deserialize(v: Value) -> Result<Self, JsonError>
```

18.6 Error type

```vesper
enum JsonError:
    UnexpectedToken(expected: string, found: string, position: u32)
    UnexpectedEof
    InvalidNumber
    InvalidUnicodeEscape
    DepthExceeded
    TypeMismatch(expected: string, found: string)
    MissingKey(key: string)
```

---

19. std.regex — Regular Expressions

19.1 Types

```vesper
pub struct Regex:
    # internal

pub struct Match:
    pub text: string
    pub start: u32
    pub end: u32
    pub groups: [Option<string>]
```

19.2 Compilation

```vesper
impl Regex:
    pub fn compile(pattern: string) -> Result<Regex, RegexError]
    pub fn is_match(self, text: string) -> bool
    pub fn find(self, text: string) -> Option<Match]
    pub fn find_all(self, text: string) -> Iterator<Match>
    pub fn captures(self, text: string) -> Option<Match]
    pub fn captures_all(self, text: string) -> Iterator<Match]
    pub fn replace(self, text: string, replacement: string) -> string
    pub fn replace_all(self, text: string, replacement: string) -> string
    pub fn split(self, text: string) -> [string]
```

19.3 Features

· Anchors: ^, $, \b, \B
· Character classes: [a-z], [^abc], \d, \w, \s
· Quantifiers: *, +, ?, {n}, {n,m}, *?, +?
· Groups: (...), (?:...), (?P<name>...)
· Backreferences: \1, \2
· Lookaround: (?=...), (?!...), (?<=...), (?<!...)
· Alternation: |
· Flags: i (case-insensitive), m (multiline), s (dotall), x (extended)

19.4 Flags

```vesper
pub const CASE_INSENSITIVE: u32 = 1 << 0
pub const MULTILINE: u32 = 1 << 1
pub const DOTALL: u32 = 1 << 2
pub const EXTENDED: u32 = 1 << 3

impl Regex:
    pub fn compile_with_flags(pattern: string, flags: u32) -> Result<Regex, RegexError>
```

19.5 Error type

```vesper
enum RegexError:
    InvalidSyntax(message: string, position: u32)
    UnmatchedGroup(position: u32)
    UnsupportedFeature(feature: string)
    TooComplex
```

---

20. std.net — Networking

20.1 TCP

```vesper
pub struct TcpListener:
    # internal

pub struct TcpStream:
    # internal

impl TcpListener:
    pub fn bind(addr: string) -> Result<TcpListener, NetError]
    pub fn accept(self) -> Result<TcpStream, NetError]
    pub fn local_addr(self) -> Result<string, NetError]
    pub fn set_nonblocking(self: TcpListener, nonblocking: bool)

impl TcpStream:
    pub fn connect(addr: string) -> Result<TcpStream, NetError]
    pub fn peer_addr(self) -> Result<string, NetError]
    pub fn local_addr(self) -> Result<string, NetError]
    pub fn set_nodelay(self: TcpStream, nodelay: bool)
    pub fn shutdown(self, how: Shutdown)
    pub fn read_timeout(self: TcpStream, d: Option<Duration>)

impl TcpStream: Reader
impl TcpStream: Writer

enum Shutdown:
    Read
    Write
    Both
```

20.2 UDP

```vesper
pub struct UdpSocket:
    # internal

impl UdpSocket:
    pub fn bind(addr: string) -> Result<UdpSocket, NetError]
    pub fn send_to(self, buf: [u8], addr: string) -> Result<u32, NetError]
    pub fn recv_from(self, buf: [u8]) -> Result<(u32, string), NetError]
    pub fn connect(self, addr: string) -> Result<unit, NetError]
    pub fn send(self, buf: [u8]) -> Result<u32, NetError]
    pub fn recv(self, buf: [u8]) -> Result<u32, NetError]
```

20.3 DNS

```vesper
pub fn resolve(hostname: string) -> Result<[string], NetError]
pub fn resolve_ipv4(hostname: string) -> Result<[string], NetError]
pub fn resolve_ipv6(hostname: string) -> Result<[string], NetError]
pub fn reverse_lookup(addr: string) -> Result<string, NetError]
```

20.4 TLS

```vesper
pub struct TlsStream:
    # internal

pub struct TlsConfig:
    pub verify_certificates: bool
    pub ca_bundle: Option<string>
    pub client_cert: Option<string>
    pub client_key: Option<string>
    pub server_name: Option<string>

impl TlsStream:
    pub fn connect(host: string, port: u16, config: TlsConfig) -> Result<TlsStream, TlsError]
    pub fn accept(stream: TcpStream, config: TlsConfig) -> Result<TlsStream, TlsError]

impl TlsStream: Reader
impl TlsStream: Writer
```

20.5 HTTP client

```vesper
pub struct HttpClient:
    # internal

pub struct Request:
    pub method: string
    pub url: string
    pub headers: {string: string}
    pub body: [u8]

pub struct Response:
    pub status: u16
    pub headers: {string: string}
    pub body: [u8]

impl HttpClient:
    pub fn new() -> HttpClient
    pub fn get(self, url: string) -> Result<Response, HttpError]
    pub fn post(self, url: string, body: [u8]) -> Result<Response, HttpError]
    pub fn request(self, req: Request) -> Result<Response, HttpError]
```

20.6 Error types

```vesper
enum NetError:
    InvalidAddress(addr: string)
    ConnectionRefused
    ConnectionReset
    Timeout
    HostUnreachable
    Other(message: string)

enum TlsError:
    CertificateError(message: string)
    HandshakeFailed(message: string)
    ProtocolError(message: string)

enum HttpError:
    Network(error: NetError)
    InvalidResponse
    TooManyRedirects
    Timeout
```

---

21. std.log — Logging

21.1 Types

```vesper
pub enum Level:
    Trace
    Debug
    Info
    Warn
    Error

pub struct Logger:
    # internal

pub struct Record:
    pub level: Level
    pub message: string
    pub fields: {string: string}
    pub timestamp: DateTime
```

21.2 Global logger

```vesper
pub fn default_logger() -> Logger
pub fn set_default_logger(l: Logger)
pub fn log(level: Level, message: string, fields: {string: string})
pub fn trace(message: string)
pub fn debug(message: string)
pub fn info(message: string)
pub fn warn(message: string)
pub fn error(message: string)
```

21.3 Logger configuration

```vesper
impl Logger:
    pub fn new() -> Logger
    pub fn with_level(self: Logger, level: Level) -> Logger
    pub fn with_writer(self: Logger, w: Writer) -> Logger
    pub fn with_format(self: Logger, format: LogFormat) -> Logger
    pub fn with_field(self: Logger, key: string, value: string) -> Logger

enum LogFormat:
    Text
    Json
    Pretty
```

21.4 Structured logging

```vesper
pub fn info_with(fields: {string: string}, message: string)
pub fn warn_with(fields: {string: string}, message: string)
pub fn error_with(fields: {string: string}, message: string)
```

---

22. std.test — Testing

22.1 Test declaration

```vesper
#[test]
fn test_addition():
    assert_eq(1 + 1, 2)

#[test]
fn test_division_by_zero():
    assert_raises(divide(1, 0), "division by zero")
```

22.2 Assertions

```vesper
pub fn assert(cond: bool, msg: string)
pub fn assert_eq<T: Comparable + Display>(actual: T, expected: T)
pub fn assert_ne<T: Comparable + Display>(actual: T, expected: T)
pub fn assert_lt<T: Comparable + Display>(a: T, b: T)
pub fn assert_gt<T: Comparable + Display>(a: T, b: T)
pub fn assert_contains(s: string, sub: string)
pub fn assert_matches(s: string, pattern: string)
pub fn assert_raises(f: fn() -> unit, msg: string)
pub fn fail(msg: string) -> !
```

22.3 Test context

```vesper
pub struct TestContext:
    pub name: string
    pub file: string
    pub line: u32

pub fn current_context() -> TestContext
```

22.4 Groups and fixtures

```vesper
pub fn group(name: string, tests: fn())

#[before_each]
fn setup():
    # called before each test

#[after_each]
fn teardown():
    # called after each test
```

22.5 Property-based tests

```vesper
pub fn for_all<T>(gen: Generator<T>, f: fn(T) -> bool)

pub struct Generator<T>:
    # internal

impl<T> Generator<T>:
    pub fn next(self) -> T
    pub fn map<U>(self, f: fn(T) -> U) -> Generator<U>
    pub fn filter(self, pred: fn(T) -> bool) -> Generator<T>

pub fn gen_int(lo: i32, hi: i32) -> Generator<i32]
pub fn gen_lists<T>(gen: Generator<T>) -> Generator<[T]]
```

---

23. std.bench — Benchmarking

```vesper
#[bench]
fn bench_sort():
    let data = gen_random_data(1000)
    bench_loop(|| {
        sort(data)
    })
```

23.1 API

```vesper
pub fn bench_loop<F: fn() -> unit>(f: F) -> BenchResult

pub struct BenchResult:
    pub iterations: u64
    pub total_time: Duration
    pub per_iteration: Duration
    pub bytes_per_second: Option<u64>
```

23.2 Helpers

```vesper
pub fn black_box<T>(x: T) -> T
pub fn gen_random_data(n: u32) -> [i32]
```

---

24. std.iter — Iterators

24.1 Overview

Iterators are lazy sequences with a next() method. The prelude provides many iterators; std.iter provides combinators.

24.2 Combinators

```vesper
pub fn map<T, U>(it: Iterator<T>, f: fn(T) -> U) -> Iterator<U>
pub fn filter<T>(it: Iterator<T>, pred: fn(T) -> bool) -> Iterator<T>
pub fn filter_map<T, U>(it: Iterator<T>, f: fn(T) -> Option<U>) -> Iterator<U>
pub fn take<T>(it: Iterator<T>, n: u32) -> Iterator<T>
pub fn take_while<T>(it: Iterator<T>, pred: fn(T) -> bool) -> Iterator<T>
pub fn skip<T>(it: Iterator<T>, n: u32) -> Iterator<T>
pub fn skip_while<T>(it: Iterator<T>, pred: fn(T) -> bool) -> Iterator<T>
pub fn chain<T>(a: Iterator<T>, b: Iterator<T>) -> Iterator<T>
pub fn enumerate<T>(it: Iterator<T>) -> Iterator<(u32, T)>
pub fn zip<A, B>(a: Iterator<A>, b: Iterator<B>) -> Iterator<(A, B)>
pub fn flatten<T>(it: Iterator<[T]>) -> Iterator<T>
pub fn flat_map<T, U>(it: Iterator<T>, f: fn(T) -> [U]) -> Iterator<U>
pub fn peekable<T>(it: Iterator<T>) -> Peekable<T>
pub fn cycle<T>(it: Iterator<T>) -> Iterator<T>       # requires T: Clone
pub fn step_by<T>(it: Iterator<T>, n: u32) -> Iterator<T>
pub fn reverse<T>(it: Iterator<T>) -> Iterator<T>

pub fn collect<T, C: FromIterator<T>>(it: Iterator<T>) -> C
pub fn count<T>(it: Iterator<T>) -> u32
pub fn sum<T: Add>(it: Iterator<T>) -> T
pub fn product<T: Mul>(it: Iterator<T>) -> T
pub fn min<T: Comparable>(it: Iterator<T>) -> Option<T>
pub fn max<T: Comparable>(it: Iterator<T>) -> Option<T>
pub fn all<T>(it: Iterator<T>, pred: fn(T) -> bool) -> bool
pub fn any<T>(it: Iterator<T>, pred: fn(T) -> bool) -> bool
pub fn find<T>(it: Iterator<T>, pred: fn(T) -> bool) -> Option<T>
pub fn position<T>(it: Iterator<T>, pred: fn(T) -> bool) -> Option<u32>
pub fn fold<A, T>(it: Iterator<T>, init: A, f: fn(A, T) -> A) -> A
pub fn reduce<T>(it: Iterator<T>, f: fn(T, T) -> T) -> Option<T>
pub fn for_each<T>(it: Iterator<T>, f: fn(T))
```

24.3 Ranges

```vesper
pub fn range(start: i32, end: i32) -> Iterator<i32>          # [start, end)
pub fn range_inclusive(start: i32, end: i32) -> Iterator<i32>  # [start, end]
pub fn range_step(start: i32, end: i32, step: i32) -> Iterator<i32>
pub fn repeat<T>(value: T) -> Iterator<T>
pub fn repeat_n<T>(value: T, n: u32) -> Iterator<T>
pub fn once<T>(value: T) -> Iterator<T>
pub fn empty<T>() -> Iterator<T]
```

---

25. std.arena — Arena Allocation

25.1 Overview

Arenas allocate many small objects and free them all at once. Ideal for compilers, parsers, and short-lived data structures.

25.2 API

```vesper
pub struct Arena:
    # internal

impl Arena:
    pub fn new() -> Arena
    pub fn with_capacity(bytes: u32) -> Arena
    pub fn alloc<T>(self: Arena, value: T) -> T          # T must be immutable
    pub fn alloc_slice<T>(self: Arena, values: [T]) -> [T]
    pub fn alloc_string(self: Arena, s: string) -> string
    pub fn bytes_allocated(self) -> u64
    pub fn reset(self: Arena)
    pub fn free(self: Arena)
```

Arena-allocated values have the lifetime of the arena. They cannot be freed individually.

25.3 Scoped arenas

```vesper
pub fn with_arena<F: fn(Arena) -> T, T>(f: F) -> T
```

The arena is automatically freed when the closure returns.

---

26. std.intern — String Interning

26.1 Overview

Interning maps strings to unique IDs. Identical strings get the same ID. Useful for compilers, parsers, and large symbol tables.

26.2 API

```vesper
pub struct Interner:
    # internal

impl Interner:
    pub fn new() -> Interner
    pub fn intern(self: Interner, s: string) -> Symbol
    pub fn lookup(self, sym: Symbol) -> string
    pub fn len(self) -> u32
    pub fn clear(self: Interner)

pub struct Symbol:
    id: u32

impl Symbol:
    pub fn id(self) -> u32
    pub fn to_string(self, interner: Interner) -> string
```

26.3 Global interner

```vesper
pub fn global() -> Interner
pub fn intern(s: string) -> Symbol
pub fn lookup(sym: Symbol) -> string
```

The global interner is thread-safe and shared across the process.

---

27. std.bits — Bit Manipulation

```vesper
pub fn popcount(x: u32) -> u32
pub fn popcount_u64(x: u64) -> u32
pub fn leading_zeros(x: u32) -> u32
pub fn trailing_zeros(x: u32) -> u32
pub fn rotate_left(x: u32, n: u32) -> u32
pub fn rotate_right(x: u32, n: u32) -> u32
pub fn reverse_bits(x: u32) -> u32
pub fn swap_bytes(x: u32) -> u32
pub fn is_power_of_two(x: u32) -> bool
pub fn next_power_of_two(x: u32) -> u32
pub fn bit_at(x: u32, i: u32) -> bool
pub fn set_bit(x: u32, i: u32) -> u32
pub fn clear_bit(x: u32, i: u32) -> u32
pub fn toggle_bit(x: u32, i: u32) -> u32
```

---

28. std.sort — Sorting

```vesper
pub fn sort<T: Comparable>(list: [T]) -> [T]
pub fn sort_by<T>(list: [T], cmp: fn(T, T) -> Ordering) -> [T]
pub fn sort_by_key<T, K: Comparable>(list: [T], key: fn(T) -> K) -> [T]
pub fn stable_sort<T: Comparable>(list: [T]) -> [T]
pub fn stable_sort_by<T>(list: [T], cmp: fn(T, T) -> Ordering) -> [T]
pub fn is_sorted<T: Comparable>(list: [T]) -> bool
pub fn binary_search<T: Comparable>(list: [T], value: T) -> Option<u32>
pub fn partition<T>(list: [T], pred: fn(T) -> bool) -> ([T], [T])
pub fn merge<T: Comparable>(a: [T], b: [T]) -> [T]
```

---

29. std.hash — Hashing

29.1 Non-cryptographic

```vesper
pub struct Fnv1a64:
    # internal

pub struct Siphash24:
    # internal

pub struct Wyhash:
    # internal

impl Fnv1a64:
    pub fn new() -> Fnv1a64
    pub fn update(self: Fnv1a64, data: [u8])
    pub fn finish(self) -> u64
```

29.2 Traits

```vesper
interface Hashable:
    fn hash(self, hasher: Hasher)

interface Hasher:
    fn write(self: Hasher, data: [u8])
    fn finish(self) -> u64
```

29.3 Convenience

```vesper
pub fn hash_u32(x: u32) -> u64
pub fn hash_u64(x: u64) -> u64
pub fn hash_string(s: string) -> u64
```

---

30. std.compress — Compression

```vesper
pub mod gzip:
    pub fn compress(data: [u8]) -> [u8]
    pub fn decompress(data: [u8]) -> Result<[u8], CompressError]

pub mod zlib:
    pub fn compress(data: [u8]) -> [u8]
    pub fn decompress(data: [u8]) -> Result<[u8], CompressError]

pub mod zstd:
    pub fn compress(data: [u8], level: u32) -> [u8]
    pub fn decompress(data: [u8]) -> Result<[u8], CompressError]

pub mod deflate:
    pub fn compress(data: [u8]) -> [u8]
    pub fn decompress(data: [u8]) -> Result<[u8], CompressError]

enum CompressError:
    InvalidHeader
    CorruptedData
    UnsupportedAlgorithm
```

---

31. std.encoding — Encodings

31.1 Base64

```vesper
pub fn base64_encode(data: [u8]) -> string
pub fn base64_decode(s: string) -> Result<[u8], EncodingError]
pub fn base64_url_encode(data: [u8]) -> string
pub fn base64_url_decode(s: string) -> Result<[u8], EncodingError]
```

31.2 Hex

```vesper
pub fn hex_encode(data: [u8]) -> string
pub fn hex_decode(s: string) -> Result<[u8], EncodingError]
pub fn hex_encode_upper(data: [u8]) -> string
```

31.3 UTF-8, UTF-16, UTF-32

```vesper
pub fn utf8_encode(s: string) -> [u8]
pub fn utf8_decode(b: [u8]) -> Result<string, EncodingError]
pub fn utf16_encode(s: string) -> [u16]
pub fn utf16_decode(b: [u16]) -> Result<string, EncodingError]
pub fn utf32_encode(s: string) -> [u32]
pub fn utf32_decode(b: [u32]) -> Result<string, EncodingError]
```

31.4 Latin-1, Windows-1252

```vesper
pub fn latin1_encode(s: string) -> [u8]
pub fn latin1_decode(b: [u8]) -> string
```

---

32. std.unicode — Unicode

32.1 Properties

```vesper
pub fn is_alphabetic(c: char) -> bool
pub fn is_numeric(c: char) -> bool
pub fn is_alphanumeric(c: char) -> bool
pub fn is_whitespace(c: char) -> bool
pub fn is_uppercase(c: char) -> bool
pub fn is_lowercase(c: char) -> bool
pub fn is_control(c: char) -> bool
pub fn is_punctuation(c: char) -> bool
pub fn is_symbol(c: char) -> bool
```

32.2 Case conversion

```vesper
pub fn to_uppercase(c: char) -> char
pub fn to_lowercase(c: char) -> char
pub fn to_titlecase(c: char) -> char
```

32.3 Normalization

```vesper
pub fn normalize_nfc(s: string) -> string
pub fn normalize_nfd(s: string) -> string
pub fn normalize_nfkc(s: string) -> string
pub fn normalize_nfkd(s: string) -> string
```

32.4 Grapheme clusters

```vesper
pub fn graphemes(s: string) -> Iterator<string>
pub fn grapheme_count(s: string) -> u32
pub fn grapheme_at(s: string, i: u32) -> Option<string>
```

---

33. std.uuid — UUID

```vesper
pub struct Uuid:
    bytes: [u8; 16]

impl Uuid:
    pub fn v4() -> Uuid                     # random
    pub fn v5(namespace: Uuid, name: string) -> Uuid    # SHA-1 based
    pub fn v7() -> Uuid                     # time-ordered
    pub fn nil() -> Uuid
    pub fn from_bytes(b: [u8; 16]) -> Uuid
    pub fn to_bytes(self) -> [u8; 16]
    pub fn from_string(s: string) -> Result<Uuid, UuidError]
    pub fn to_string(self) -> string
    pub fn to_hyphenated(self) -> string
    pub fn to_simple(self) -> string
    pub fn version(self) -> u8
```

---

34. std.error — Error Handling

34.1 Overview

Utilities for working with errors across modules.

34.2 Type-erased errors

```vesper
pub struct Error:
    message: string
    cause: Option<Error>
    source: Option<string>

impl Error:
    pub fn new(message: string) -> Error
    pub fn with_cause(self, cause: Error) -> Error
    pub fn with_source(self, source: string) -> Error
    pub fn message(self) -> string
    pub fn cause(self) -> Option<Error]
    pub fn chain(self) -> Iterator<Error>
```

34.3 Conversion

```vesper
pub fn to_error<E: Display>(e: E) -> Error
pub fn box_error<E: Display>(e: E) -> Error
```

34.4 Context

```vesper
pub fn context<T, E>(result: Result<T, E>, msg: string) -> Result<T, Error]
```

---

35. std.ffi — Foreign Function Interface

35.1 Overview

The FFI allows Vesper code to call C functions and be called from C. This is inherently unsafe and requires explicit opt-in.

35.2 Declaring foreign functions

```vesper
@extern("c")
fn malloc(size: u32) -> *u8

@extern("c")
fn free(ptr: *u8)

@extern("libm")
fn cos(x: f64) -> f64
```

35.3 Unsafe blocks

```vesper
unsafe:
    let p = malloc(1024)
    if p != null:
        # use p
        free(p)
```

Unsafe blocks are required for:

· Dereferencing raw pointers.
· Calling C functions.
· Transmuting types.

35.4 Raw pointers

```vesper
pub struct *T:
    # opaque raw pointer

impl<T> *T:
    pub unsafe fn null() -> *T
    pub unsafe fn is_null(self) -> bool
    pub unsafe fn read(self) -> T
    pub unsafe fn write(self: *T, value: T)
    pub unsafe fn offset(self: *T, n: i32) -> *T
    pub unsafe fn cast<U>(self) -> *U
```

Raw pointers are only usable inside unsafe blocks.

35.5 C ABI types

```vesper
pub type CInt = i32
pub type CLong = i64
pub type CChar = i8
pub type CSize = u64
pub type CVoid = unit
```

---

36. std.debug — Debugging

```vesper
pub fn dump<T: Debug>(value: T) -> string
pub fn dump_value<T: Debug>(value: T)
pub fn backtrace() -> [StackFrame]
pub fn print_backtrace()

pub struct StackFrame:
    pub function: string
    pub file: string
    pub line: u32
```

---

37. std.prof — Profiling

```vesper
pub fn start_cpu_profile(path: string) -> Result<Profile, IOError]
pub fn start_memory_profile(path: string) -> Result<Profile, IOError]

pub struct Profile:
    # internal

impl Profile:
    pub fn stop(self) -> Result<unit, IOError]
```

---

38. std.trace — Tracing

```vesper
pub struct Span:
    # internal

pub fn start_span(name: string) -> Span
pub fn end_span(span: Span)

impl Span:
    pub fn set_attribute(self: Span, key: string, value: string)
    pub fn add_event(self: Span, name: string)
    pub fn record_error(self: Span, err: Error)

pub fn with_span<F: fn() -> T, T>(name: string, f: F) -> T
```

Spans are exported via OpenTelemetry Protocol (OTLP) when configured.

---

39. Conventions and Guarantees

39.1 Stability

Public API of std follows SemVer:

· Patch releases — bug fixes, no API changes.
· Minor releases — additive changes only.
· Major releases — breaking changes with a deprecation cycle.

39.2 Thread safety

Functions are thread-safe unless documented otherwise. Types marked Send can be sent between processes; types marked Sync can be shared by reference.

39.3 Determinism

Functions that read external state (time, randomness, file system) are clearly marked. Deterministic mode of ARC VM forces these to reproducible values.

39.4 Documentation

Every public item has:

· A one-line summary.
· A longer description.
· At least one example.
· Notes on error conditions.
· Notes on performance, if relevant.

39.5 Testing

Every public function is tested:

· Unit tests for logic.
· Property tests for invariants.
· Fuzz tests for parsing functions.
· Integration tests for I/O functions.

39.6 Benchmarking

Performance-critical functions have benchmarks committed to the repository. Regressions are caught in CI.

39.7 Deprecation

Deprecated APIs are marked with @deprecated("use X instead") and kept for at least one minor version. Removal happens in the next major version.

39.8 Platform support

std targets:

· Tier 1: Linux amd64, macOS arm64, Windows amd64.
· Tier 2: Linux arm64, macOS amd64.
· Tier 3: Other platforms supported on a best-effort basis.

Functions that are unavailable on a platform return an error, not panic.

---

40. Versioning and Stability

40.1 Version scheme

std is versioned independently from the compiler, but the versions are coupled: each compiler version ships with a matching std version.

40.2 Compatibility

std version X.Y is compatible with compiler version X.Y and any later X.* compiler.

40.3 Feature gates

Experimental APIs are gated behind feature flags:

```toml
[dependencies]
std = { version = "0.1", features = ["experimental-net"] }
```

Without the feature, the API is unavailable.

40.4 Migration

When a major version introduces breaking changes, vspm provides vspm migrate to update code automatically. Migration guides are published with each major release.

---

Appendix A: Module Summary

Module Purpose Stability
std.io Readers, writers, streams Stable
std.text Strings, parsing, formatting Stable
std.collections Vec, Map, Set, BTree, Deque, Heap Stable
std.math Integer and float math, BigInt, Rational Stable
std.time Clocks, Duration, DateTime Stable
std.fs File system Stable
std.path Path manipulation Stable
std.process Subprocesses Stable
std.os OS-specific operations Stable
std.env Environment (deprecated) Deprecated
std.sync Mutex, RwLock, atomics, channels Stable
std.crypto Hashing, ciphers, signatures Stable
std.rand Random numbers Stable
std.json JSON parsing and serialization Stable
std.regex Regular expressions Stable
std.net TCP, UDP, DNS, TLS, HTTP client Stable
std.log Structured logging Stable
std.test Testing framework Stable
std.bench Benchmarking Stable
std.iter Iterator combinators Stable
std.arena Arena allocation Stable
std.intern String interning Stable
std.bits Bit manipulation Stable
std.sort Sorting algorithms Stable
std.hash Non-cryptographic hashing Stable
std.compress gzip, zlib, zstd Stable
std.encoding Base64, hex, UTF Stable
std.unicode Unicode properties Stable
std.uuid UUID generation Stable
std.error Error utilities Stable
std.ffi Foreign function interface Experimental
std.debug Debugging helpers Stable
std.prof Profiling Experimental
std.trace Distributed tracing Experimental

---

Appendix B: Change Log

Version 0.1 (draft)

· Initial specification.
· 35 modules covering I/O, text, collections, math, time, fs, path, process, os, env, sync, crypto, rand, json, regex, net, log, test, bench, iter, arena, intern, bits, sort, hash, compress, encoding, unicode, uuid, error, ffi, debug, prof, trace.
· All APIs are safe by default; unsafe blocks required for FFI.
· Constant-time crypto operations by default.
· Deterministic mode supported in time, random, and process modules.
· Deprecation policy and SemVer compatibility.

---

End of specification.

---

Философская заметка

Стандартная библиотека — это лицо языка. Пользователь судит о Vesper не по красоте синтаксиса, а по тому, насколько приятно писать read_file, HashMap::new(), json.parse. Если std интуитивна, последовательна и полна, язык будет жить. Если она дырявая, медленная, или полна странных компромиссов — никакая красота синтаксиса не спасёт.

Поэтому std — не «остаток» после проектирования языка, а его первая и главная часть. Язык проектируется вместе со стандартной библиотекой, а не до неё.

Главные правила:

1. Всё, что можно выразить в языке, выражается в std, а не в компиляторе. Компилятор знает только о примитивах. Всё остальное — библиотека.
2. Каждый модуль — самостоятелен и тестируется отдельно. std.json не зависит от std.net. std.text не зависит от std.fs.
3. Никаких скрытых зависимостей. Если функция читает время — это видно из имени и сигнатуры.
4. Безопасность — не опция. Функции, работающие с памятью, криптографией или сетью, безопасны по умолчанию.
5. Производительность — часть контракта. Функция с O(n²) сложностью не имеет права называться find.
