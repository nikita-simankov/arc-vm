The Vesper Programming Language Specification

Version 0.1 (Draft)
Target: ARC Virtual Machine
Editors: Vesper Language Design Team

---

Table of Contents

1. Introduction
2. Notation
3. Source Code Representation
4. Lexical Elements
5. Constants
6. Variables
7. Types
8. Properties of Types and Values
9. Blocks and Scoping
10. Declarations
11. Expressions
12. Statements
13. Built-in Functions
14. Modules and Import
15. Program Initialization and Execution
16. Errors and Panics
17. Concurrency
18. Compilation to ARC Assembly

---

1. Introduction

This document specifies the Vesper programming language. Vesper is a statically typed, compiled language targeting the ARC Virtual Machine. It draws syntax and semantics from Python (indentation, readability), Go (explicit types, methods with receivers, interfaces, channels), and Elixir (pattern matching, pipe operator, atoms, immutability by default).

The key goals of the language, in priority order, are:

1. Safety. No raw pointers, no undefined behavior, no unchecked memory access. Vesper inherits ARC's safety guarantees by construction.
2. Readability. Code is read far more often than it is written. Syntax is optimized for comprehension.
3. Expressiveness. Pattern matching, pipelines, and algebraic data types allow concise expression of complex ideas.
4. Performance. Compilation to ARC assembly is direct; no hidden allocations or runtime magic.
5. Familiarity. Anyone coming from Go, Python, or Elixir should feel at home within hours.

The implementation is not required by this specification. Any compiler that produces ARC assembly conforming to this document is a valid Vesper implementation.

---

2. Notation

The syntax is specified using Extended Backus–Naur Form (EBNF):

```
Production  = production_name "=" [ Expression ] "." .
Expression  = Alternative { "|" Alternative } .
Alternative = Term { Term } .
Term        = production_name | token [ "…" token ] | Group | Option | Repetition .
Group       = "(" Expression ")" .
Option      = "[" Expression "]" .
Repetition  = "{" Expression "}" .
```

Productions are expressions constructed from terms and the following operators, in increasing precedence:

```
|   alternation
()  grouping
[]  option (0 or 1 times)
{}  repetition (0 to n times)
```

Lowercase production names are used to identify lexical (terminal) tokens. Non-terminals are in CamelCase. Lexical tokens are enclosed in double quotes "" or back quotes ``.

The form a … b represents the set of characters from a through b as alternatives.

A link to a non-terminal in the text refers to the production of that name.

---

3. Source Code Representation

3.1 Characters

Vesper source is Unicode text encoded in UTF-8. The text is not canonicalized, so a single accented code point is distinct from the same character constructed from combining an accent and a letter; those are treated as two code points. For simplicity, this document will use the unqualified term character to refer to a Unicode code point in the source text.

Each code point is distinct; for instance, uppercase and lowercase letters are different characters.

3.2 Letters and Digits

The underscore character _ (U+005F) is considered a letter.

```
letter      = unicode_letter | "_" .
decimal_digit = "0" … "9" .
binary_digit  = "0" | "1" .
octal_digit   = "0" … "7" .
hex_digit     = "0" … "9" | "A" … "F" | "a" … "f" .
```

3.3 Indentation

Vesper uses indentation to delimit blocks, as in Python. Indentation is significant at the lexical level:

· A block begins after a line ending with a colon :.
· Indentation must be consistent within a block, using 4 spaces.
· Tabs are not permitted for indentation. Using a tab character in leading whitespace is a lexical error.
· A dedent ends the current block.
· Blank lines and comment-only lines are ignored for indentation purposes.

Optionally, a block may be explicitly terminated by the keyword end at the same indentation level as the construct that opened it. This is discouraged for short blocks but may improve readability for long ones.

3.4 Semicolons

Vesper does not require semicolons at the end of statements. The lexer inserts an implicit end-of-statement token at each newline that ends a logical line, following the same rules as Go:

· A newline terminates a statement if the last token on the line is an identifier, a literal, a closing delimiter (), ], }), or one of the keywords break, continue, return, end, pass.
· A newline does not terminate a statement if the line ends inside an open delimiter or with a binary operator.

Explicit semicolons ; may be used to separate statements on a single line.

---

4. Lexical Elements

4.1 Comments

```
LineComment      = "#" { unicode_char } .
BlockComment     = "#*" { unicode_char } "*#" .
```

A line comment begins with # and continues to the end of the line. A block comment begins with #* and ends with *#. Block comments do not nest.

Comments are treated as whitespace for the purposes of parsing.

4.2 Tokens

Tokens form the vocabulary of the language. There are four classes:

· Identifiers
· Keywords
· Operators and punctuation
· Literals

Whitespace, formed from spaces, tabs, carriage returns, and newlines, is ignored except as it separates tokens or affects indentation. Other whitespace characters (e.g., non-breaking spaces) are illegal.

4.3 Identifiers

```
identifier = letter { letter | unicode_digit } .
```

Identifiers name program entities such as variables, types, functions, and methods. An identifier is a sequence of one or more letters and digits. The first character must be a letter.

```
foo
_private
camelCase
snake_case
π
```

By convention, names use snake_case for variables and functions, PascalCase for types and interfaces. Names beginning with an underscore are unexported (visible only within the defining module); see §10.1.

4.4 Keywords

The following keywords are reserved and may not be used as identifiers:

```
as        break     case      catch     chan
const     continue  defer     elif      else
end       enum      false     fn        for
if        import    in        interface let
loop      match     module    None      Ok
pub       receive   return    select    self
spawn     struct    true      try       use
var       while     with      yield
```

4.5 Operators and Punctuation

```
+    -    *    /    %    **    
&    |    ^    ~    <<   >>
&&   ||   !
==   !=   <    <=   >    >=
=    +=   -=   *=   /=   %=
&=   |=   ^=   <<=  >>=
<-   |>   ..   ..=  ?    .
,    :    ;    (    )    [    ]
{    }    #    @    _    $
```

The following character sequences terminate statements automatically: ,, ;, (, ), [, ], {, }.

4.6 Integer Literals

```
int_lit     = decimal_lit | binary_lit | octal_lit | hex_lit .
decimal_lit = "0" | ( "1" … "9" ) [ [ "_" ] decimal_digits ] .
binary_lit  = "0b" ( binary_digit | "_" ) { binary_digit | "_" } .
octal_lit   = "0o" ( octal_digit | "_" ) { octal_digit | "_" } .
hex_lit     = "0x" ( hex_digit | "_" ) { hex_digit | "_" } .
```

Underscores are permitted as digit separators and are ignored.

```
42
1_000_000
0b1010_1010
0o755
0xDEAD_BEEF
```

4.7 Floating-Point Literals

```
float_lit   = decimals "." [ decimals ] [ exponent ]
            | decimals exponent
            | "." decimals [ exponent ] .
decimals    = decimal_digit { decimal_digit } .
exponent    = ( "e" | "E" ) [ "+" | "-" ] decimals .
```

```
3.14159
.5
1e10
6.022e23
1_000.000_1
```

4.8 String Literals

```
string_lit  = raw_string_lit | interpreted_string_lit .
raw_string_lit         = "r" `"` { unicode_char } `"` .
interpreted_string_lit = `"` { unicode_char | escape } `"` .
```

A raw string literal, prefixed with r, contains no escape sequences. An interpreted string literal may contain escape sequences:

```
\a    U+0007 alert
\b    U+0008 backspace
\f    U+000C form feed
\n    U+000A line feed
\r    U+000D carriage return
\t    U+0009 tab
\v    U+000B vertical tab
\\    backslash
\"    double quote
\{    left brace (escaped for interpolation)
\uXXXX  Unicode escape (exactly 4 hex digits)
\UXXXXXXXX  Unicode escape (exactly 8 hex digits)
```

4.9 String Interpolation

String literals support interpolation of expressions enclosed in braces {…}:

```
name = "Alice"
print("Hello, {name}!")
```

An expression inside {…} is evaluated at runtime. To output a literal brace, escape it as \{ or \}.

4.10 Rune Literals

```
rune_lit = "'" ( unicode_char | escape ) "'" .
```

A rune literal represents a single Unicode code point:

```
'a'
'π'
'\n'
'\u0041'
```

4.11 Atoms

```
atom_lit = ":" identifier .
```

An atom is a named constant. Two atoms with the same name are identical. Atoms are commonly used as lightweight tags:

```
:ok
:error
:not_found
:user_created
```

Atoms are values of type Atom. Atom values are compared by identity.

4.12 Boolean Literals

```
true
false
```

4.13 Nil Literal

The literal None denotes the absence of a value. It is the sole member of the type None and can be assigned to any Option<T> and to any interface.

---

5. Constants

A constant is a name bound to a compile-time value. Constants must be initialized with expressions that are themselves constant.

```
ConstDecl = "const" identifier [ Type ] "=" Expression .
```

Examples:

```
const PI = 3.14159
const MAX_SIZE: i32 = 1024
const GREETING = "Hello, World!"
const ENABLED = true
```

The types permitted for constants are: boolean, numeric (integer and floating-point), string, and atoms. Constants are implicitly of their underlying type unless specified.

Constant expressions may involve other constants, arithmetic, string concatenation, and comparison:

```
const A = 10
const B = 20
const SUM = A + B          # 30
const NAME = "Ves" + "per"  # "Vesper"
```

---

6. Variables

A variable is a storage location bound to a name. Vesper distinguishes between immutable and mutable variables.

```
VarDecl     = ( "let" | "var" ) identifier [ Type ] "=" Expression .
ShortDecl   = identifier ":=" Expression .
```

6.1 Immutable Variables (let)

let introduces an immutable binding. Once initialized, its value cannot be reassigned. Attempting to assign to an immutable binding is a compile-time error.

```
let x = 10
let name = "Alice"
x = 20             # error: x is immutable
```

6.2 Mutable Variables (var)

var introduces a mutable binding:

```
var counter = 0
counter += 1
counter = 100
```

6.3 Short Declaration (:=)

Inside a function, a short declaration introduces a new mutable variable with type inferred from the initializer:

```
x := 10            # equivalent to: var x i32 = 10
name := "Alice"
```

:= may not be used at package level, and may not re-declare a name in the same scope.

6.4 Shadowing

An inner scope may declare a variable with the same name as an outer scope. This is called shadowing. The inner binding is used until the inner scope ends, then the outer binding is visible again.

```
let x = 10
if true:
    let x = 20
    print(x)       # 20
print(x)           # 10
```

6.5 Destructuring

Variables may be declared by destructuring a tuple, struct, or other composite value:

```
let (a, b) = (1, 2)
let Point(x, y) = origin
let (first, ..rest) = [1, 2, 3, 4]
```

See §11.9 for pattern-matching rules.

---

7. Types

A type determines the set of values and the operations that may be performed on those values.

```
Type      = TypeName | TypeLit | "(" Type ")" .
TypeName  = identifier | QualifiedIdent .
TypeLit   = ArrayType | TupleType | MapType | StructType | InterfaceType
          | FunctionType | ChannelType | OptionType | ResultType
          | AtomicType | PointerType .
```

7.1 Boolean Type

```
bool
```

The boolean type has exactly two values: true and false.

7.2 Numeric Types

Vesper provides the following numeric types:

Type Size Range
i8 8 bits −128 … 127
i16 16 bits −32 768 … 32 767
i32 32 bits −2³¹ … 2³¹−1
i64 64 bits −2⁶³ … 2⁶³−1
u8 8 bits 0 … 255
u16 16 bits 0 … 65 535
u32 32 bits 0 … 2³²−1
u64 64 bits 0 … 2⁶⁴−1
f32 32 bits IEEE-754 binary32
f64 64 bits IEEE-754 binary64

Numeric literals are untyped and take the type required by their context. In the absence of a context, an integer literal takes type i32, and a floating-point literal takes type f64.

7.3 String Type

```
string
```

A string is an immutable, UTF-8 encoded sequence of bytes. Strings are values: assigning a string copies the reference, but the underlying data cannot be modified.

The built-in function len returns the number of bytes in a string.

7.4 Array Type

```
ArrayType = "[" Expression "]" Type .
```

An array is a fixed-length sequence of values of the same type. The length is part of the type.

```
[i32]              # variable-length list of i32
[i32; 10]          # array of exactly 10 i32 (rare, mostly for FFI)
```

Vesper primarily uses variable-length arrays (called lists). Fixed-size arrays with a length are used only in rare FFI contexts.

A list is accessed by index, with bounds checking performed automatically. Reading or writing past the end of a list raises the BOUNDS exception (see §16).

7.5 Tuple Type

```
TupleType = "(" [ TypeList ] ")" .
```

A tuple is a fixed-length sequence of values of possibly different types.

```
(i32, string)
(f64, f64, f64)
()
```

Tuples are value types: assigning a tuple copies all its elements.

7.6 Map Type

```
MapType = "{" Type ":" Type "}" .
```

A map is a hash table with keys of one type and values of another.

```
{i32: string}
{string: [i32]}
```

The key type must be comparable with == (see §11.4). Common key types are i32, u32, string, and Atom.

7.7 Struct Type

```
StructType = "struct" identifier [ ":" ] Indent { FieldDecl } .
FieldDecl  = identifier ":" Type .
```

A struct is a record with named fields. Fields may be of any type.

```
struct Point:
    x: f64
    y: f64

struct Person:
    name: string
    age: u32
    email: Option<string>
```

Struct types may embed other struct types by declaring a field whose name is the type name:

```
struct Animal:
    name: string

struct Dog:
    Animal
    breed: string
```

Embedding promotes the fields and methods of the embedded type to the outer type. d.name accesses d.Animal.name directly, and any method defined on Animal is also a method of Dog (unless shadowed by a method defined directly on Dog).

Structs are value types. Assigning a struct copies all its fields.

7.8 Interface Type

```
InterfaceType = "interface" identifier ":" Indent { MethodSig } .
MethodSig     = "fn" MethodName Signature .
```

An interface specifies a set of methods that a type must have. Any type that has all these methods implicitly satisfies the interface.

```
interface Shape:
    fn area(self) -> f64
    fn perimeter(self) -> f64
```

A type implements the interface as soon as it defines methods with matching signatures. No explicit declaration is required. This is identical to Go's structural typing.

The empty interface Any is predefined and is satisfied by every type:

```
interface Any:
    # no methods
```

7.9 Function Type

```
FunctionType = "fn" Signature .
Signature    = "(" [ ParameterList ] ")" [ "->" Type ] .
ParameterList = Parameter { "," Parameter } .
Parameter    = [ identifier ":" ] Type [ "=" Expression ] .
```

A function type describes a callable value:

```
fn (i32, i32) -> i32
fn (string) -> unit
fn () -> bool
```

7.10 Channel Type

```
ChannelType = "Chan" "<" Type ">" .
```

A channel is a typed conduit for values between processes (see §17).

```
Chan<i32>
Chan<string>
Chan<(u32, string)>
```

7.11 Option Type

```
OptionType = "Option" "<" Type ">" .
```

An Option<T> value is either Some(v) for some value v of type T, or None.

```
Option<i32>
Option<string>
Option<Point>
```

7.12 Result Type

```
ResultType = "Result" "<" Type "," Type ">" .
```

A Result<T, E> value is either Ok(v) for a value v of type T, or Err(e) for an error e of type E.

```
Result<i32, string>
Result<Point, IOError>
```

7.13 Pointer Type

Vesper does not have raw pointers. The only pointer-like construct is the reference receiver used in method declarations, indicated by (p: T):

```
fn (p: Point) translate(dx: f64, dy: f64):
    p.x += dx
    p.y += dy
```

Inside the method, p refers to the caller's value, and mutations are visible to the caller. This is exactly Go's pointer receiver, but the pointer is opaque: it cannot be reassigned, cast to an integer, or compared with anything other than == and !=.

7.14 Atomic Types

```
AtomicType = "Atomic" "<" Type ">" .
```

An Atomic<T> is a mutable value that supports lock-free atomic operations. The type parameter must be a numeric type or bool.

```
Atomic<i32>
Atomic<u64>
Atomic<bool>
```

See §17.4.

7.15 Named Types

A type declaration introduces a named type:

```
TypeDecl = "struct" | "enum" | "interface" | "type" .
```

For aliases:

```
type Celsius = f64
type UserID = u32
```

A named type is distinct from its underlying type. UserID and u32 are different types, even though they share the same representation. Values of one cannot be assigned to variables of the other without an explicit conversion.

7.16 Enum Types

```
EnumDecl = "enum" identifier ":" Indent { EnumVariant } .
EnumVariant = identifier [ "(" [ FieldList ] ")" ] .
```

An enum is a sum type: a value is exactly one of the named variants.

```
enum Shape:
    Circle(radius: f64)
    Rectangle(w: f64, h: f64)
    Triangle(a: f64, b: f64, c: f64)
```

Variants may carry zero or more named fields.

---

8. Properties of Types and Values

8.1 Type Identity

Two types are identical if they are defined the same way. Named types are identical to themselves only.

8.2 Assignability

A value x of type V is assignable to a variable of type T if:

· V and T are identical.
· V and T have identical underlying types, and at least one is not a named type.
· T is an interface and V implements T.
· T is Option<U> and V is U (auto-wrapping) or None.
· T is Result<U, E> and V is U (auto-wrapping to Ok) or an error value of type E (auto-wrapping to Err).

8.3 Comparability

Values of the following types may be compared with == and !=:

· Boolean
· Numeric
· String
· Atom
· Tuple of comparable types
· Struct of comparable fields
· Interface (compares dynamic type and value)
· Option of comparable type
· None

Lists, maps, and channels are not comparable with ==. Use explicit comparison functions.

8.4 Method Sets

The method set of a type T is the set of methods declared with receiver of type T. The method set of *T (reference) includes all methods declared with receiver T or *T.

An interface is satisfied by T if the interface's methods are a subset of T's method set.

---

9. Blocks and Scoping

A block is a sequence of declarations and statements, delimited by indentation:

```
Block = Indent { Statement } .
```

The scope of an identifier declared inside a block extends from the point of declaration to the end of the innermost enclosing block.

Blocks are introduced by:

· Function bodies
· if, elif, else
· for, while, loop
· match, case
· try, catch
· Bare blocks (a block used for scoping only)

The scope of a top-level declaration extends to the end of the module.

---

10. Declarations

10.1 Declaration Scope

A declaration binds an identifier to a program entity. Declarations may be:

· Exported (visible outside the module), prefixed with pub.
· Unexported (visible only within the module), with no prefix.
· Local (visible only within the enclosing block).

10.2 Constant Declarations

```
ConstDecl = "const" identifier [ Type ] "=" Expression .
```

10.3 Variable Declarations

```
VarDecl   = ( "let" | "var" ) identifier [ Type ] "=" Expression .
```

10.4 Type Declarations

```
TypeDecl  = "struct" identifier ":" Indent { FieldDecl }
          | "enum" identifier ":" Indent { EnumVariant }
          | "interface" identifier ":" Indent { MethodSig }
          | "type" identifier "=" Type .
```

10.5 Function Declarations

```
FuncDecl = "fn" identifier Signature ":" Indent { Statement } .
```

Examples:

```
fn add(a: i32, b: i32) -> i32:
    a + b

fn greet(name: string = "World"):
    print("Hello, {name}!")

fn identity<T>(x: T) -> T:
    x
```

The last expression in a function body is the return value if it is not terminated by ; and if the function declares a return type. Otherwise, return must be used explicitly.

10.6 Method Declarations

A method is a function with a receiver — the first parameter, enclosed in its own parentheses before the function name.

```
MethodDecl = "fn" Receiver identifier Signature ":" Indent { Statement } .
Receiver   = "(" identifier [ ":" ] TypeName ")" .
```

The receiver clause is Go-style. The identifier is the receiver name; the type is the receiver type.

Two forms exist:

· Value receiver: (p Point) — the method receives a copy. Mutations are local to the method.
· Reference receiver: (p: Point) — the method receives a reference to the caller's value. Mutations are visible to the caller.

Examples:

```
struct Point:
    x: f64
    y: f64

fn (p Point) length() -> f64:
    sqrt(p.x * p.x + p.y * p.y)

fn (p: Point) translate(dx: f64, dy: f64):
    p.x += dx
    p.y += dy
```

Method declarations must be in the same module as the type's declaration. Declaring a method on a type defined in another module is a compile-time error.

Methods are invoked with dot syntax:

```
let p = Point(x: 3.0, y: 4.0)
print(p.length())          # 5.0
p.translate(1.0, 2.0)      # p.x is now 4.0
```

This is syntactic sugar for a direct function call:

```
p.length()          ≡ length(p)
p.translate(1, 2)   ≡ translate(p, 1, 2)
```

10.7 Interface Satisfaction

A type T satisfies an interface I if T's method set includes all methods of I. This is checked at compile time when T is assigned to a variable of type I, passed as an argument to a parameter of type I, or returned as a result of type I.

No explicit declaration is required. This is identical to Go's implicit interface satisfaction.

10.8 Module Declarations

```
ModuleDecl = "module" QualifiedIdent .
```

A module declaration appears once at the top of the file:

```
module geometry.point
```

See §14.

---

11. Expressions

11.1 Operands

```
Operand = Literal | OperandName | "(" Expression ")" | Call | IndexExpr
        | Selector | TypeAssert | Comprehension | BlockExpr .
```

11.2 Arithmetic Operators

Applied to numeric operands:

```
+    sum
-    difference
*    product
/    quotient
%    remainder
**   exponentiation
```

+ is also defined for strings (concatenation) and for lists (concatenation).

11.3 Bitwise and Shift Operators

```
&    bitwise AND
|    bitwise OR
^    bitwise XOR
~    bitwise NOT
<<   left shift
>>   arithmetic right shift
```

11.4 Comparison Operators

```
==   equal
!=   not equal
<    less than
<=   less than or equal
>    greater than
>=   greater than or equal
```

For strings, comparison is lexicographic on bytes.

11.5 Logical Operators

```
&&   logical AND (short-circuit)
||   logical OR (short-circuit)
!    logical NOT
```

11.6 Function and Method Calls

```
Call = PrimaryExpr "(" [ ArgumentList ] ")" .
ArgumentList = Argument { "," Argument } .
Argument = Expression | NamedArgument .
NamedArgument = identifier ":" Expression .
```

Named arguments are matched to parameters by name:

```
fn create_user(name: string, age: u32, active: bool = true): ...

create_user(name: "Alice", age: 30)
create_user(age: 25, name: "Bob", active: false)
```

Positional and named arguments may not be mixed.

11.7 Index Expressions

```
IndexExpr = PrimaryExpr "[" Expression "]" .
```

Applied to lists, maps, and strings. Bounds are checked at runtime for lists and strings. Missing keys in maps return the zero value of the value type.

```
xs[0]
m["key"]
s[3]
```

11.8 Slice Expressions

```
SliceExpr = PrimaryExpr "[" [ Expression ] ":" [ Expression ] [ ":" Expression ] "]" .
```

Applies to lists and strings. Returns a subrange:

```
xs[2:5]       # elements at index 2, 3, 4
xs[:5]        # elements from 0 up to 5
xs[2:]        # elements from 2 to end
xs[:]         # copy of the list
```

11.9 Pattern Matching

Patterns appear in match, let, and case contexts.

```
Pattern = Literal | identifier | TuplePattern | ListPattern | StructPattern
        | EnumPattern | TypePattern | Wildcard | RestPattern
        | "(" Pattern ")" .
```

· Literal patterns match exact values: 42, "hello", :ok, true.
· Identifier patterns bind the matched value to the identifier.
· Tuple patterns destructure tuples: (a, b).
· List patterns match list shapes: [], [x], [x, ..rest], [first, second, ..rest].
· Struct patterns destructure structs: Point(x, y), Person(name, age, ..).
· Enum patterns match variants: Circle(r), Rectangle(w, h).
· Type patterns match on dynamic type: x.(i32), x.(Point).
· Wildcard _ matches anything without binding.
· Rest .. matches the remainder in list and struct patterns.

Guards may be attached to patterns with if:

```
match x:
    n if n > 100 -> print("big")
    n if n > 0   -> print("positive")
    _            -> print("non-positive")
```

11.10 Pipe Operator

The pipe operator |> passes the value on the left as the first argument to the function on the right:

```
x |> f(y)     ≡ f(x, y)
```

Pipes chain naturally and are left-associative:

```
data |> filter(pred) |> map(transform) |> reduce(init, combine)
```

11.11 Comprehensions

```
Comprehension = "[" Expression "for" Pattern "in" Expression
                { "for" Pattern "in" Expression }
                { "if" Expression } "]" .
```

Produces a list by iterating over one or more sources and filtering with optional conditions:

```
[x * x for x in 0..10]
[x for x in 0..20 if x % 2 == 0]
[(x, y) for x in 0..3 for y in 0..3]
```

Comprehensions may also be written for maps:

```
{k: v * 2 for (k, v) in original}
```

11.12 Type Assertions

```
TypeAssert = PrimaryExpr "." "(" Type ")" .
```

Attempts to interpret an interface value as a concrete type. The result is an Option<T>:

```
match shape.(Circle):
    Some(c) -> print("radius: {c.radius}")
    None    -> print("not a circle")
```

11.13 Selector Expressions

```
Selector = PrimaryExpr "." identifier .
```

Selects a field or method:

```
point.x
point.length()
person.name
```

For an embedded field, the selector may omit the intermediate name:

```
dog.name       # same as dog.Animal.name
dog.describe() # same as dog.Animal.describe()
```

11.14 Receive Expression

```
ReceiveExpr = "<-" Expression .
```

Receives a value from a channel or mailbox (see §17):

```
value = <- ch
```

11.15 Block Expression

A block may be used as an expression. The value of the block is the value of its last expression:

```
let x = if cond:
    compute_a()
else:
    compute_b()
```

---

12. Statements

12.1 Expression Statements

Any expression may be used as a statement. The value is discarded.

```
print("hello")
x + 1
```

12.2 Assignment Statements

```
Assignment = ( identifier | PrimaryExpr ) AssignOp Expression .
AssignOp   = "=" | "+=" | "-=" | "*=" | "/=" | "%="
           | "&=" | "|=" | "^=" | "<<=" | ">>=" .
```

Compound assignments +=, -=, etc., are shortcuts for x = x op y. The left-hand side must be assignable (an identifier or a mutable location).

```
x = 10
x += 5
xs[2] = 42
p.x = 3.0
```

Multiple assignment is permitted:

```
x, y = y, x
```

12.3 If Statements

```
IfStmt = "if" Expression ":" Block
         { "elif" Expression ":" Block }
         [ "else" ":" Block ] .
```

```
if x > 0:
    print("positive")
elif x < 0:
    print("negative")
else:
    print("zero")
```

An if statement may be used as an expression:

```
let label = if x > 0: "positive" else: "non-positive"
```

12.4 For Statements

```
ForStmt = "for" ( RangeClause | Pattern "in" Expression ) ":" Block .
RangeClause = [ InitStmt ] ";" [ Condition ] ";" [ PostStmt ] .
```

Iterate over a list, tuple, map, channel, or range:

```
for x in xs:
    print(x)

for (k, v) in m:
    print("{k} = {v}")

for i in 0..10:        # 0 to 9
    print(i)

for i in 0..=10:       # 0 to 10
    print(i)
```

C-style loop:

```
for i := 0; i < 10; i += 1:
    print(i)
```

Loop over a channel:

```
for v in ch:
    print(v)
```

The loop ends when the channel is closed.

12.5 While Statements

```
WhileStmt = "while" Expression ":" Block .
```

```
while i < 10:
    print(i)
    i += 1
```

while is provided for readability; it is equivalent to for with only a condition.

12.6 Loop Statements

```
LoopStmt = "loop" ":" Block .
```

An infinite loop. Exit via break or return.

```
loop:
    line = read_line()
    if line == "quit":
        break
    print(line)
```

12.7 Break and Continue

```
break [ Label ]
continue [ Label ]
```

break exits the innermost enclosing loop (or loop, while, for, select). continue skips to the next iteration.

12.8 Return Statements

```
ReturnStmt = "return" [ Expression ] .
```

If a function declares a return type, return must be followed by a value of a compatible type.

12.9 Pass Statement

```
pass
```

A no-op. Used as a placeholder where a statement is required but none is wanted.

12.10 Match Statements

```
MatchStmt = "match" Expression ":" Indent { MatchCase } .
MatchCase = Pattern [ "if" Expression ] "->" ( Expression | Block ) .
```

Matches the subject against patterns in order. The first matching case's body executes.

```
match code:
    200 -> print("OK")
    404 -> print("Not Found")
    n if n >= 500 -> print("Server error: {n}")
    _ -> print("Unknown: {code}")
```

A match may be used as an expression; all arms must then produce values of the same type.

12.11 Try-Catch Statements

```
TryStmt = "try" ":" Block "catch" Pattern ":" Block .
```

Catches ARC VM exceptions (see §16):

```
try:
    x = risky_compute()
    print(x)
catch e:
    print("caught: {e}")
```

12.12 Defer Statements

```
DeferStmt = "defer" Call .
```

Schedules a function call to run when the enclosing function returns, whether normally or via an exception.

```
fn process_file(path: string):
    f = open(path)
    defer close(f)
    # ...
```

Multiple defers run in LIFO order.

12.13 Spawn Statements

```
SpawnStmt = "spawn" ( Call | "do" ":" Block ) .
```

Starts a new process (see §17):

```
spawn worker(arg)

spawn do:
    print("in background")
```

12.14 Send Statements

```
SendStmt = Expression "<-" Expression .
```

Sends a value to a channel or a mailbox:

```
ch <- 42
pid <- {self(), :ping}
```

12.15 Select Statements

```
SelectStmt = "select" ":" Indent { SelectCase } .
SelectCase = ( ReceiveClause | SendClause | AfterClause ) ":" Block .
ReceiveClause = Pattern "=" "<-" Expression .
SendClause    = Expression "<-" Expression .
AfterClause   = "after" "(" Duration ")" .
```

Waits for one of several channel operations to become ready:

```
select:
    v = ch1 <-:
        print("ch1: {v}")
    v = ch2 <-:
        print("ch2: {v}")
    after(5.seconds):
        print("timeout")
```

---

13. Built-in Functions

The following functions are predefined and may be used without importing anything.

Function Signature Description
len fn(len: T) -> u32 Length of a string, list, map, or channel capacity.
cap fn(cap: Chan<T>) -> u32 Capacity of a channel.
append fn(xs: [T], x: T) -> [T] Returns a new list with x appended.
concat fn(xs: [T], ys: [T]) -> [T] Returns a new list with ys appended to xs.
slice fn(xs: [T], lo: u32, hi: u32) -> [T] Returns a subrange.
keys fn(m: {K: V}) -> [K] Keys of a map.
values fn(m: {K: V}) -> [V] Values of a map.
get fn(m: {K: V}, k: K, default: V) -> V Map lookup with a default.
has fn(m: {K: V}, k: K) -> bool Whether the map contains the key.
print fn(x: Any) Prints a value followed by a newline.
println fn(x: Any) Alias for print.
assert fn(cond: bool, msg: string) Raises assert_failed if the condition is false.
panic fn(msg: string) Aborts execution with an error.
min fn(a: T, b: T) -> T Smaller of two values.
max fn(a: T, b: T) -> T Larger of two values.
abs fn(x: T) -> T Absolute value.
sqrt fn(x: f64) -> f64 Square root.
sin, cos, tan fn(x: f64) -> f64 Trigonometric functions.
floor, ceil, round fn(x: f64) -> f64 Rounding.
int_to_string fn(x: i32) -> string Convert an integer to a string.
string_to_int fn(s: string) -> Result<i32, string> Parse an integer.
char_at fn(s: string, i: u32) -> u8 Byte at the given index.
substring fn(s: string, lo: u32, hi: u32) -> string Substring.
self fn() -> PID Returns the PID of the current process.
close fn(ch: Chan<T>) Closes a channel.
recv fn(ch: Chan<T>) -> Option<T> Receives one value; returns None if closed.
send fn(ch: Chan<T>, v: T) Sends a value.

Additional functions are provided by the standard library modules std.io, std.text, std.math, std.crypto, std.net, std.time, and others.

---

14. Modules and Import

14.1 Module Declaration

Every source file begins with a module declaration:

```
module geometry.point
```

The module path may contain dots to indicate a hierarchy. It must match the file's location within the project.

14.2 Import Declaration

```
ImportDecl = "import" ImportPath [ "." "{" ImportList "}" ] .
ImportPath = QualifiedIdent .
ImportList = ImportName { "," ImportName } .
ImportName = identifier | "{" identifier "," identifier "}" .
```

Examples:

```
import std.io
import std.text.{ split, trim, lines }
import geometry.point
import geometry.point.{ Point, distance }
```

The last form imports specific symbols from a module into the current file's namespace.

14.3 Export Rules

A top-level declaration is exported if it is marked pub. Exported identifiers are visible to any module that imports the declaring module.

```
pub struct Point:
    x: f64
    y: f64

pub fn (p Point) length() -> f64:
    sqrt(p.x * p.x + p.y * p.y)

fn internal_helpers():
    # not visible outside this module
```

14.4 Module Initialization

Each module may contain an init function that is called once before main:

```
fn init():
    print("module loaded")
```

Modules are initialized in dependency order, once per program.

---

15. Program Initialization and Execution

15.1 Entry Point

Every executable program has a main function in a main module:

```
module main

fn main():
    print("Hello, World!")
```

15.2 Order of Execution

1. All imported modules are initialized in dependency order. Each module's init function is called exactly once.
2. main.main is called.
3. When main returns, the program exits with code 0.
4. If main panics, the program exits with a non-zero code.

15.3 The _start Symbol

The compiler generates a _start symbol in the produced ARC assembly that:

1. Calls main.main.
2. Exits with code 0 on success.

This symbol is the point of entry for the ARC VM.

---

16. Errors and Panics

16.1 Result and Option

The idiomatic way to handle errors in Vesper is to use Result<T, E> and Option<T>. These are not exceptions: they are values. A function that may fail should return a Result and the caller should inspect it.

```
fn divide(a: i32, b: i32) -> Result<i32, string>:
    if b == 0:
        Err("division by zero")
    else:
        Ok(a / b)

fn main():
    match divide(10, 2):
        Ok(v)  -> print("result: {v}")
        Err(e) -> print("error: {e}")
```

16.2 The ? Operator

Inside a function returning Result<T, E>, the ? operator unwraps an Ok value or returns early with the Err value:

```
fn compute() -> Result<i32, string>:
    a = divide(10, 2)?
    b = divide(a, 3)?
    Ok(a + b)
```

If divide(10, 2) returns Err(e), the entire function returns Err(e). Otherwise, a is bound to the inner value.

An analogous operator exists for Option: inside a function returning Option<T>, ? unwraps Some(v) or returns None.

16.3 Panics

A panic is a fatal error that aborts the program. Panics are reserved for programming errors — invariant violations, impossible states, bugs — not for expected error conditions.

```
panic("unreachable")
assert(x > 0, "x must be positive")
```

16.4 Exceptions

ARC VM provides structured exceptions (see the ARC architecture document). Vesper exposes a limited try/catch construct for interacting with code that may raise an ARC exception — for example, arithmetic overflow, out-of-bounds access, or null dereference.

```
try:
    x = risky_operation()
catch e:
    print("caught exception: {e}")
```

Regular application errors should use Result, not exceptions. try/catch should be used only at boundaries where ARC exceptions are expected.

---

17. Concurrency

Vesper provides lightweight concurrency via processes, channels, and mailboxes.

17.1 Processes

A process is a lightweight unit of execution, mapped 1-to-1 to an ARC VM thread (which is, in turn, mapped to a Go goroutine). Processes share memory only through explicitly shared objects (see §17.4).

To start a process:

```
spawn worker(arg1, arg2)

spawn do:
    print("in a new process")
```

spawn returns a process identifier (PID):

```
pid = spawn worker()
```

17.2 Channels

A channel is a typed conduit between processes.

```
let ch: Chan<i32> = chan()               # unbuffered
let ch: Chan<i32> = chan(buffer: 10)     # buffered, capacity 10
```

Send and receive:

```
ch <- 42                # send
v = <- ch               # receive, v is i32
```

If the channel is unbuffered, a send blocks until a receiver is ready, and vice versa. If the channel is buffered, sends block only when the buffer is full.

Receive from a channel using a loop:

```
for v in ch:
    print(v)
```

The loop terminates when the channel is closed and drained.

Close a channel:

```
close(ch)
```

Sending on a closed channel is a panic. Receiving from a closed channel returns immediately with the zero value or, in the option-returning form, None.

17.3 Mailboxes

Every process has a mailbox. Messages are sent to a process's PID and received with the receive keyword.

```
fn worker():
    loop:
        match receive():
            {sender, :ping} -> sender <- :pong
            {_, :stop}      -> break

let pid = spawn worker()

pid <- {self(), :ping}
match receive():
    :pong -> print("got pong")

pid <- {self(), :stop}
```

self() returns the current process's PID. receive() blocks until a message arrives in the mailbox.

17.4 Atomic Operations

Atomic values support lock-free operations:

```
let counter = Atomic<i32>(0)

counter.add(1)                      # atomic increment
counter.store(100)                  # atomic store
let v = counter.load()              # atomic load
counter.compare_and_swap(100, 200)  # CAS: if value == 100, set to 200
```

Atomic operations translate directly to ARC's atomic_add, atomic_xchg, atomic_cmpxchg, atomic_load, and atomic_store instructions.

17.5 Select

select waits for one of several channel operations to complete:

```
select:
    v = ch1 <-:
        print("from ch1: {v}")
    v = ch2 <-:
        print("from ch2: {v}")
    after(5.seconds):
        print("timed out")
```

If multiple cases are ready, one is chosen non-deterministically. If none is ready and an after clause exists, the after clause fires once the duration elapses.

17.6 Deterministic Mode

When the ARC VM is run in deterministic mode, process scheduling follows a fixed round-robin policy with a fixed time quantum, and all timing-related operations are virtualized. In this mode, the same program with the same input produces bit-identical output on every run.

---

18. Compilation to ARC Assembly

This section describes how Vesper constructs map to ARC assembly. It is informative but not normative: any compilation strategy that preserves observable behavior is valid.

18.1 Data Layout

· Integers of 8, 16, and 32 bits use one ARC register.
· Integers of 64 bits use a register pair (r1:r0).
· Floating-point values of 32 bits use one register; 64-bit values use a pair.
· Booleans use one register (0 or 1).
· Strings are pointers to ARC string objects.
· Lists are pointers to ARC array objects with a length header.
· Maps are pointers to hash table objects.
· Structs are laid out with fields at fixed offsets, aligned to their natural size.
· Tuples are laid out like structs with fields named 0, 1, …
· Interfaces are fat pointers: a pair of (vtable_ptr, data_ptr).
· Option<T> is a tagged value: a tag (0 for None, 1 for Some) plus the payload.
· Result<T, E> is a tagged value: a tag (0 for Ok, 1 for Err) plus a union of payloads.
· Channels are pointers to ARC channel objects.
· Atomic<T> is a pointer to a specially-aligned cell.

18.2 Function Compilation

Each Vesper function becomes one ARC label:

Vesper:

```
fn add(a: i32, b: i32) -> i32:
    a + b
```

ARC:

```asm
add:
    enter #0
    add   r0, r0, r1
    leave
    ret
```

The first four parameters are passed in r0–r3; additional parameters are passed on the data stack.

The return value is placed in r0 (or r1:r0 for 64-bit values).

18.3 Method Compilation

A method is compiled as a function whose first argument is the receiver:

Vesper:

```
fn (p Point) length() -> f64:
    sqrt(p.x * p.x + p.y * p.y)
```

ARC:

```asm
point_length:
    enter #0
    push  r4
    mov   r4, r0
    load.d r0, [r4]         ; p.x
    load.d r1, [r4 + 8]     ; p.y
    fmul  r2, r0, r0
    fmul  r3, r1, r1
    fadd  r0, r2, r3
    fsqrt r0, r0
    pop   r4
    leave
    ret
```

A value receiver passes the struct by value (the compiler may pass a pointer if the struct is large, but semantics are copy-on-write). A reference receiver passes the pointer directly.

18.4 Interface Compilation

Each interface value is a fat pointer: (vtable_ptr, data_ptr).

The compiler generates one vtable per (concrete type, interface) pair:

```asm
.rodata
circle_shape_vtable:
    .word circle_area
    .word circle_perimeter

rect_shape_vtable:
    .word rect_area
    .word rect_perimeter
```

A method call on an interface value loads the vtable, indexes into it, and calls the resulting function pointer:

```asm
    load r1, [s + 0]         ; r1 = vtable_ptr
    load r2, [r1 + 0]        ; r2 = area method
    load r0, [s + 4]         ; r0 = data_ptr
    call r2                  ; indirect call
```

18.5 Control Flow

· if/elif/else compiles to cmp and conditional jumps.
· for over a list compiles to an index loop with load.
· while compiles to a cmp + conditional jump back to the loop head.
· match compiles to a sequence of comparisons and conditional branches.

18.6 Pattern Matching

Each match case becomes a block of code:

```asm
    ; match code { 200 -> ... 404 -> ... _ -> ... }
    cmp  r0, #200
    jeq  .case_200
    cmp  r0, #404
    jeq  .case_404
    jmp  .case_default
.case_200:
    ; case body
    jmp  .end_match
.case_404:
    ; case body
    jmp  .end_match
.case_default:
    ; default body
.end_match:
```

18.7 Pipe Operator

x |> f(y) compiles to a regular function call with x as the first argument:

```asm
    mov r0, x
    mov r1, y
    call f
```

18.8 Concurrency Primitives

· spawn f(args) → mov r0, f; mov r1, args; syscall SYS_thread_create
· ch <- v → mov r0, ch; mov r1, v; syscall SYS_chan_send
· v = <- ch → mov r0, ch; syscall SYS_chan_recv; mov v, r0
· close(ch) → mov r0, ch; syscall SYS_chan_close
· self() → syscall SYS_self

18.9 Errors and Panics

· panic("msg") → writes msg to stderr and exits with a non-zero code.
· assert(cond, msg) → cmp cond, #0; jne .ok; panic(msg); .ok:
· try/catch → SYS_exc_register for the handler, trap for the protected block, resume for the exit.

18.10 String Interpolation

"Hello, {name}!" is expanded into a sequence of string operations:

1. Convert name to a string (via to_string if necessary).
2. Concatenate the literal and the string.
3. Result is a new string.

18.11 Constants

Constants are placed in .rodata:

```asm
.rodata
PI:      .word 0x40490FDB    ; 3.14159 as f32
GREETING: .string "Hello"
```

---

Appendix A: Grammar Summary

A condensed EBNF of the full grammar.

```
SourceFile    = ModuleDecl { ImportDecl } { TopLevelDecl } .

ModuleDecl    = "module" QualifiedIdent .
QualifiedIdent = identifier { "." identifier } .

ImportDecl    = "import" QualifiedIdent [ "." "{" ImportList "}" ] .

TopLevelDecl  = ConstDecl | VarDecl | TypeDecl | FuncDecl | MethodDecl .

ConstDecl     = "const" identifier [ ":" Type ] "=" Expression .
VarDecl       = ( "let" | "var" ) identifier [ ":" Type ] "=" Expression .

TypeDecl      = "struct" identifier ":" BlockFields
              | "enum" identifier ":" BlockVariants
              | "interface" identifier ":" BlockMethods
              | "type" identifier "=" Type .

FuncDecl      = "fn" identifier Signature ":" Block .
MethodDecl    = "fn" Receiver identifier Signature ":" Block .
Receiver      = "(" identifier [ ":" ] TypeName ")" .
Signature     = [ TypeParams ] "(" [ ParameterList ] ")" [ "->" Type ] .

Statement     = ExpressionStmt | Assignment | IfStmt | ForStmt | WhileStmt
              | LoopStmt | MatchStmt | TryStmt | ReturnStmt | BreakStmt
              | ContinueStmt | DeferStmt | SpawnStmt | SendStmt | SelectStmt
              | PassStmt .

Expression    = UnaryExpr | BinaryExpr | Call | IndexExpr | SliceExpr
              | Selector | TypeAssert | Comprehension | BlockExpr
              | IfExpr | MatchExpr | PipeExpr .

Pattern       = Literal | identifier | TuplePattern | ListPattern
              | StructPattern | EnumPattern | TypePattern | "_" | ".." .

Type          = TypeName | ArrayType | TupleType | MapType | StructType
              | InterfaceType | FunctionType | ChannelType | OptionType
              | ResultType | AtomicType | "(" Type ")" .
```

---

Appendix B: Change Log

Version 0.1 (draft)

· Initial specification.
· Go-style methods with receivers (fn (p Point) length() -> f64:), replacing the earlier Rust-style impl blocks.
· Interfaces instead of traits, with implicit satisfaction (Go-style).
· Embedding of structs for composition (Go-style).
· Any as a universal interface.
· Type assertions with x.(T).
· Option<T> and Result<T, E> with the ? operator.
· Python-style indentation.
· Elixir-style pattern matching, pipe operator |>, atoms.
· Processes, channels, and mailboxes as the concurrency model.
