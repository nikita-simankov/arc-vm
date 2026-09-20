The Vesper Language Server Protocol Specification

Version 0.1 (Draft)
Server binary: vs-lsp
Protocol: LSP 3.17
Transport: stdio (primary), TCP (secondary)

---

Table of Contents

1. Introduction
2. Architecture Overview
3. Transport and Lifecycle
4. Capabilities
5. Text Synchronization
6. Diagnostics
7. Completion
8. Hover
9. Signature Help
10. Go-to-Definition and Related
11. Find References
12. Document Symbols
13. Workspace Symbols
14. Rename and Prepare Rename
15. Code Actions
16. Code Lens
17. Formatting
18. Semantic Tokens
19. Inlay Hints
20. Folding Ranges
21. Selection Ranges
22. Document Highlight
23. Call Hierarchy
24. Type Hierarchy
25. Workspace Folders and Configuration
26. Custom Extensions
27. Performance and Caching
28. Error Handling
29. Security
30. Version Compatibility

---

1. Introduction

The Vesper Language Server (vs-lsp) provides IDE support for the Vesper programming language (see the Vesper Language Specification). It implements the Language Server Protocol version 3.17, exposing type information, refactorings, diagnostics, and navigation to any LSP-compatible editor (VS Code, Neovim, Emacs, Helix, Zed, etc.).

This document specifies:

· The set of LSP methods vs-lsp implements.
· The behaviour of each method in the context of Vesper.
· Custom Vesper-specific extensions to the protocol.
· Performance and correctness requirements.

This document is normative. It describes what any conforming Vesper language server must do. An implementation may extend but must not contradict this specification.

---

2. Architecture Overview

2.1 Components

```
+-----------------+        JSON-RPC over stdio       +----------------+
|                 | <------------------------------> |                |
|   Editor/LSP    |                                  |    vs-lsp      |
|    Client       |                                  |    Server      |
|                 |                                  |                |
+-----------------+                                  +-------+--------+
                                                             |
                                            +----------------+----------------+
                                            |                                 |
                                   +--------v--------+               +--------v---------+
                                   |  Frontend       |               |  Analysis Engine |
                                   |  (parse, lex)   |               |  (types, symbols)|
                                   +--------+--------+               +--------+---------+
                                            |                                 |
                                            +----------------+----------------+
                                                             |
                                                    +--------v--------+
                                                    |   Workspace     |
                                                    |   Index         |
                                                    |  (in-memory)    |
                                                    +-----------------+
```

2.2 Request Flow

1. Client sends initialize.
2. Server responds with capabilities.
3. Client sends initialized.
4. Client sends textDocument/didOpen for each open file.
5. Server parses each file, indexes symbols, and produces diagnostics.
6. Client sends incremental changes via textDocument/didChange.
7. Server re-parses and re-diagnoses, notifying the client via textDocument/publishDiagnostics.
8. Client sends requests (hover, definition, completion, etc.).
9. Server responds.

2.3 Threading Model

The server is single-threaded from the protocol's point of view, but internally uses a worker pool:

· Main thread: reads and writes JSON-RPC messages.
· Worker pool: parses, type-checks, and indexes files.
· Request handlers: run on worker threads, but responses are queued back to the main thread to preserve ordering.

Requests that modify state (rename, formatting, codeAction with edits) acquire a write lock on the workspace. Read-only requests (hover, completion) use a read lock and may run in parallel.

2.4 Documents and Workspace

The server maintains:

· Open documents: files the client has opened. Content is authoritative.
· Closed documents: files the server reads from disk. Content may be stale.
· Workspace index: a global symbol table for all files in the workspace.

The index is updated incrementally: opening or editing a file triggers a re-index of that file only.

---

3. Transport and Lifecycle

3.1 Transport

Primary transport: stdio.

· Reads: stdin.
· Writes: stdout.
· Logs: stderr.

The server must handle Content-Length headers exactly as specified by LSP 3.17.

Secondary transport (optional): TCP on a port specified by the client via --port.

3.2 Lifecycle Requests

3.2.1 initialize

Request:

```json
{
  "processId": 12345,
  "rootUri": "file:///home/user/project",
  "capabilities": { ... },
  "initializationOptions": {
    "vesper": {
      "compilerPath": "/usr/local/bin/vsc",
      "arcAsPath": "/usr/local/bin/arc-as",
      "arcVmPath": "/usr/local/bin/arc-vm",
      "enableInlayHints": true,
      "enableSemanticTokens": true,
      "maxDiagnosticsPerFile": 200,
      "diagnosticsOnChange": "debounced"
    }
  },
  "workspaceFolders": [
    { "uri": "file:///home/user/project", "name": "project" }
  ]
}
```

Response:

```json
{
  "capabilities": {
    "textDocumentSync": {
      "openClose": true,
      "change": 2,
      "save": { "includeText": false }
    },
    "completionProvider": {
      "triggerCharacters": [".", ":", "|", "(", "<"],
      "resolveProvider": true
    },
    "hoverProvider": true,
    "signatureHelpProvider": {
      "triggerCharacters": ["(", ","],
      "retriggerCharacters": [")"]
    },
    "definitionProvider": true,
    "typeDefinitionProvider": true,
    "implementationProvider": true,
    "referencesProvider": true,
    "documentSymbolProvider": true,
    "workspaceSymbolProvider": true,
    "codeActionProvider": {
      "codeActionKinds": [
        "quickfix",
        "refactor",
        "refactor.extract",
        "refactor.inline",
        "refactor.rewrite",
        "source.organizeImports",
        "source.fixAll"
      ]
    },
    "codeLensProvider": { "resolveProvider": true },
    "documentFormattingProvider": true,
    "documentRangeFormattingProvider": true,
    "documentOnTypeFormattingProvider": {
      "firstTriggerCharacter": "\n",
      "moreTriggerCharacter": [":", "}"]
    },
    "renameProvider": { "prepareProvider": true },
    "foldingRangeProvider": true,
    "selectionRangeProvider": true,
    "documentHighlightProvider": true,
    "semanticTokensProvider": {
      "legend": {
        "tokenTypes": [
          "namespace", "type", "class", "enum", "interface",
          "struct", "typeParameter", "parameter", "variable",
          "property", "enumMember", "event", "function",
          "method", "macro", "keyword", "modifier", "comment",
          "string", "number", "regexp", "operator", "decorator",
          "atom"
        ],
        "tokenModifiers": [
          "declaration", "definition", "readonly", "static",
          "deprecated", "abstract", "async", "modification",
          "documentation", "defaultLibrary", "mutable",
          "exported", "embedded"
        ]
      },
      "full": { "delta": true },
      "range": false
    },
    "inlayHintProvider": {
      "resolveProvider": true
    },
    "callHierarchyProvider": true,
    "typeHierarchyProvider": true,
    "workspace": {
      "workspaceFolders": {
        "supported": true,
        "changeNotifications": true
      }
    },
    "serverInfo": {
      "name": "vs-lsp",
      "version": "0.1.0"
    }
  }
}
```

3.2.2 initialized

Notification from client, no response. Server may begin sending requests (e.g., workspace/configuration, client/registerCapability).

3.2.3 shutdown

Request. Server flushes caches, stops worker threads, responds with null.

3.2.4 exit

Notification. Server exits with code 0 if shutdown was previously received, else 1.

3.3 Cancellation

All requests support cancellation via $/cancelRequest. The server must handle cancellation gracefully: long-running operations check the cancellation token at least every 100ms.

---

4. Capabilities

vs-lsp must implement:

Capability Requirement
textDocumentSync Full and incremental
completionProvider With resolveProvider: true
hoverProvider Full markdown
signatureHelpProvider Yes
definitionProvider Yes
typeDefinitionProvider Yes
implementationProvider Yes
referencesProvider Yes
documentSymbolProvider Hierarchical
workspaceSymbolProvider Yes
codeActionProvider Yes
codeLensProvider Yes
documentFormattingProvider Yes
documentRangeFormattingProvider Yes
renameProvider With prepareProvider: true
foldingRangeProvider Yes
selectionRangeProvider Yes
documentHighlightProvider Yes
semanticTokensProvider Full + delta
inlayHintProvider Yes
callHierarchyProvider Yes
typeHierarchyProvider Yes

vs-lsp may implement:

Capability Optional
colorProvider Optional (for hex colors in strings)
linkedEditingRangeProvider Optional (for matching braces)
monikerProvider Optional
inlineValueProvider Optional
diagnosticProvider Pull-model diagnostics (see §6.3)

---

5. Text Synchronization

5.1 textDocument/didOpen

Notifies the server that a document has been opened.

```json
{
  "textDocument": {
    "uri": "file:///home/user/project/src/main.vsp",
    "languageId": "vesper",
    "version": 1,
    "text": "module main\n\nfn main():\n    print(\"Hello\")\n"
  }
}
```

Server behaviour:

1. Store the text with its version.
2. Parse the file, produce diagnostics, and send them via textDocument/publishDiagnostics.
3. Index the file into the workspace.
4. If a .vsp file was previously open with old content, discard the old content.

5.2 textDocument/didChange

Incremental changes. Each change contains a range and replacement text.

```json
{
  "textDocument": {
    "uri": "file:///home/user/project/src/main.vsp",
    "version": 2
  },
  "contentChanges": [
    {
      "range": {
        "start": { "line": 3, "character": 4 },
        "end":   { "line": 3, "character": 10 }
      },
      "rangeLength": 6,
      "text": "world"
    }
  ]
}
```

Server behaviour:

1. Apply changes to the in-memory document.
2. Re-parse the changed region (or the whole file if the change spans multiple top-level constructs).
3. Re-run diagnostics after a debounce interval (default 200ms).
4. Re-publish diagnostics for the file and any dependents.

5.3 textDocument/didSave

Notifies the server that a file was saved.

```json
{
  "textDocument": { "uri": "file:///home/user/project/src/main.vsp" },
  "text": "…"
}
```

Server behaviour:

1. Run full diagnostics (not debounced).
2. If text is present, use it as authoritative; otherwise read from disk.
3. Trigger a workspace re-index of dependents if the file exports changed.

5.4 textDocument/didClose

Notifies the server that a document was closed.

Server behaviour:

1. Remove the file from open documents.
2. Read the file from disk for future operations.
3. Clear diagnostics for the file (send empty diagnostics).
4. Keep the file in the workspace index (do not remove).

5.5 textDocument/willSave, willSaveWaitUntil

Optional. vs-lsp does not implement these. Clients should not rely on them.

---

6. Diagnostics

6.1 Push-Model Diagnostics

vs-lsp uses the push model: after every parse or type-check, diagnostics are sent via textDocument/publishDiagnostics.

```json
{
  "uri": "file:///home/user/project/src/main.vsp",
  "version": 2,
  "diagnostics": [
    {
      "range": {
        "start": { "line": 5, "character": 8 },
        "end":   { "line": 5, "character": 12 }
      },
      "severity": 1,
      "code": "VSP-E001",
      "codeDescription": { "href": "https://vesper.dev/errors/VSP-E001" },
      "source": "vsc",
      "message": "undefined variable `x`",
      "relatedInformation": [
        {
          "location": {
            "uri": "file:///home/user/project/src/other.vsp",
            "range": {
              "start": { "line": 10, "character": 4 },
              "end":   { "line": 10, "character": 5 }
            }
          },
          "message": "similar variable `xs` declared here"
        }
      ],
      "tags": [1],
      "data": {
        "suggestions": ["xs", "x1"]
      }
    }
  ]
}
```

6.2 Diagnostic Sources

Source Trigger
Lexer errors Invalid tokens, indentation errors
Parser errors Syntax errors
Type errors Type mismatches, missing methods
Name resolution errors Undefined identifiers
Import errors Unresolved imports
Unused code warnings Unused variables, functions, imports
Style warnings Naming conventions, shadowing

6.3 Pull-Model Diagnostics (Optional)

If the client supports textDocument/diagnostic, the server may also expose pull-model diagnostics. This is recommended for large workspaces where push-model updates cause too much traffic.

6.4 Diagnostic Codes

All diagnostic codes follow the pattern VSP-<letter><number>:

Prefix Category
VSP-E Error
VSP-W Warning
VSP-I Information
VSP-H Hint

Example codes:

Code Message
VSP-E001 Undefined variable
VSP-E002 Type mismatch
VSP-E003 Undefined function
VSP-E004 Wrong number of arguments
VSP-E005 Method not found
VSP-E006 Interface not satisfied
VSP-E007 Immutable binding cannot be reassigned
VSP-E008 Non-exhaustive match
VSP-E009 Unreachable code
VSP-W001 Unused variable
VSP-W002 Unused import
VSP-W003 Shadowing outer binding
VSP-W004 Deprecated API
VSP-W005 Lossy numeric conversion
VSP-I001 Implicit type conversion
VSP-H001 Consider using let instead of var

6.5 Debounce

Diagnostics on didChange are debounced by 200ms by default. Configurable via initializationOptions.vesper.diagnosticsOnChange:

· "debounced" (default) — 200ms debounce.
· "immediate" — send on every change.
· "save" — send only on save.

---

7. Completion

7.1 textDocument/completion

Provides completion items at the cursor position.

Request:

```json
{
  "textDocument": { "uri": "file:///home/user/project/src/main.vsp" },
  "position": { "line": 10, "character": 8 },
  "context": {
    "triggerKind": 1,
    "triggerCharacter": "."
  }
}
```

Response:

```json
{
  "isIncomplete": false,
  "items": [
    {
      "label": "length",
      "kind": 2,
      "detail": "fn (p Point) length() -> f64",
      "documentation": {
        "kind": "markdown",
        "value": "Returns the Euclidean length of the point.\n\n```vesper\nlet p = Point(x: 3.0, y: 4.0)\nprint(p.length())  # 5.0\n```"
      },
      "sortText": "0_length",
      "filterText": "length",
      "insertText": "length()",
      "insertTextFormat": 2,
      "textEdit": {
        "range": {
          "start": { "line": 10, "character": 6 },
          "end":   { "line": 10, "character": 8 }
        },
        "newText": "length()"
      },
      "additionalTextEdits": [],
      "commitCharacters": ["(", ".", " "],
      "data": {
        "kind": "method",
        "receiverType": "Point"
      }
    }
  ]
}
```

7.2 Completion Triggers

Trigger Context
. Member access on a value
: Start of type annotation, atom, or named argument
| Pipe operator completion
( Function call, argument completion
< Generic type argument
{ String interpolation, map literal
@ Reserved for future use

7.3 Completion Kinds

LSP kind Vesper entities
Text Keywords, atoms
Method Struct methods
Function Free functions
Constructor Struct literals, enum variants
Field Struct fields
Variable Local variables
Class Struct types
Interface Interface types
Enum Enum types
EnumMember Enum variants
Module Module names
TypeParameter Generic type parameters
Constant const declarations
Keyword Language keywords

7.4 Completion Sorting

Items are sorted by:

1. Exact match at cursor (sortText prefix 0_).
2. Prefix match (1_).
3. Fuzzy match (2_).
4. Everything else (3_).

Sort priority is determined by:

· Whether the item is in scope.
· Whether the item is imported.
· Whether the item is a struct field matching the expected type.
· Recent use (recency cache maintained per document).

7.5 completionItem/resolve

Client may request additional details for a completion item.

Request:

```json
{
  "label": "length",
  "kind": 2,
  "data": {
    "kind": "method",
    "receiverType": "Point"
  }
}
```

Response: completion item with documentation and detail filled in.

The server must be able to resolve items from their data field. If data is missing or unrecognized, the item is returned unchanged.

7.6 Snippet Support

If the client declares completionItem.snippetSupport, insert texts may contain placeholders:

```
Point(x: ${1:0.0}, y: ${2:0.0})
```

The server must always provide a non-snippet fallback when snippetSupport is false.

---

8. Hover

8.1 textDocument/hover

Provides information about the symbol at the cursor.

Request:

```json
{
  "textDocument": { "uri": "file:///home/user/project/src/main.vsp" },
  "position": { "line": 10, "character": 8 }
}
```

Response:

```json
{
  "contents": {
    "kind": "markdown",
    "value": "```vesper\nfn (p Point) length() -> f64\n```\n\n---\n\nReturns the Euclidean length of the point.\n\n**Example**\n\n```vesper\nlet p = Point(x: 3.0, y: 4.0)\nprint(p.length())  # 5.0\n```"
  },
  "range": {
    "start": { "line": 10, "character": 6 },
    "end":   { "line": 10, "character": 12 }
  }
}
```

8.2 Hover Content by Entity

Entity Content
Variable Type + declaration location
Function Signature + doc comment
Method Signature + receiver type + doc comment
Struct Definition + field list
Interface Definition + method list
Enum Definition + variant list
Atom Type Atom + usage hint
Type alias Underlying type
Keyword Short description from language spec

8.3 Hover for Types in Error

If the symbol cannot be resolved, the server returns null. It does not guess.

---

9. Signature Help

9.1 textDocument/signatureHelp

Provides parameter information for a function call in progress.

Request:

```json
{
  "textDocument": { "uri": "file:///home/user/project/src/main.vsp" },
  "position": { "line": 15, "character": 22 },
  "context": {
    "triggerKind": 2,
    "triggerCharacter": ",",
    "isRetrigger": false
  }
}
```

Response:

```json
{
  "signatures": [
    {
      "label": "fn create_user(name: string, age: u32, active: bool = true)",
      "documentation": {
        "kind": "markdown",
        "value": "Creates a new user with the given attributes."
      },
      "parameters": [
        { "label": [11, 24], "documentation": "The user's name." },
        { "label": [26, 34], "documentation": "The user's age." },
        { "label": [36, 57], "documentation": "Whether the user is active." }
      ]
    }
  ],
  "activeSignature": 0,
  "activeParameter": 1
}
```

9.2 Overloads

If multiple overloads exist (rare in Vesper, but possible with generics), all are returned. The client picks the best match based on the current argument index and type.

9.3 Retrigger

If the client sends isRetrigger: true, the server may reuse cached signature information for the current call expression.

---

10. Go-to-Definition and Related

10.1 textDocument/definition

Request:

```json
{
  "textDocument": { "uri": "file:///home/user/project/src/main.vsp" },
  "position": { "line": 10, "character": 8 }
}
```

Response: A Location or array of Location:

```json
{
  "uri": "file:///home/user/project/src/geometry/point.vsp",
  "range": {
    "start": { "line": 12, "character": 0 },
    "end":   { "line": 12, "character": 34 }
  }
}
```

If the symbol is defined in a compiled .arc file for which no source is available, the server returns null. Future versions may support navigation into disassembled ARC code.

10.2 textDocument/typeDefinition

Navigates to the type definition of the symbol at the cursor. For example, if the cursor is on a variable p of type Point, this navigates to struct Point.

10.3 textDocument/implementation

Navigates to implementations of an interface method or a method dispatch site.

For an interface method, returns all concrete methods that implement it.

10.4 textDocument/declaration

Navigates to the declaration site of a symbol. For most Vesper entities, declaration and definition are the same location; for functions declared in an interface and implemented in a struct, they differ.

10.5 Precision Requirements

The server must return exact ranges. Ambiguous results are not permitted.

---

11. Find References

11.1 textDocument/references

Request:

```json
{
  "textDocument": { "uri": "file:///home/user/project/src/geometry/point.vsp" },
  "position": { "line": 12, "character": 10 },
  "context": { "includeDeclaration": true }
}
```

Response:

```json
[
  {
    "uri": "file:///home/user/project/src/geometry/point.vsp",
    "range": { "start": { "line": 12, "character": 0 }, "end": { "line": 12, "character": 34 } }
  },
  {
    "uri": "file:///home/user/project/src/main.vsp",
    "range": { "start": { "line": 10, "character": 8 }, "end": { "line": 10, "character": 12 } }
  }
]
```

11.2 Scope of Search

The server searches:

1. All open documents.
2. All files indexed in the workspace.

Symbols in external modules (standard library) are not searched unless they are part of the workspace.

11.3 includeDeclaration

If true, the declaration site is included. Otherwise, only usage sites are returned.

11.4 Performance

For workspaces under 100 files, reference search completes in under 200ms. For larger workspaces, the server uses the workspace index and returns partial results with the possibility of a follow-up. Clients are notified via $/progress if the search takes more than 1 second.

---

12. Document Symbols

12.1 textDocument/documentSymbol

Returns a hierarchical tree of symbols in a document.

Response:

```json
[
  {
    "name": "Point",
    "kind": 23,
    "range": { "start": {"line": 5, "character": 0}, "end": {"line": 8, "character": 0} },
    "selectionRange": { "start": {"line": 5, "character": 7}, "end": {"line": 5, "character": 12} },
    "children": [
      {
        "name": "x",
        "kind": 8,
        "range": { "start": {"line": 6, "character": 4}, "end": {"line": 6, "character": 12} },
        "selectionRange": { "start": {"line": 6, "character": 4}, "end": {"line": 6, "character": 5} }
      },
      {
        "name": "length",
        "kind": 6,
        "detail": "fn (p Point) length() -> f64",
        "range": { "start": {"line": 10, "character": 0}, "end": {"line": 11, "character": 20} },
        "selectionRange": { "start": {"line": 10, "character": 12}, "end": {"line": 10, "character": 18} }
      }
    ]
  }
]
```

12.2 Symbol Kinds

Vesper entity LSP symbol kind
Module Namespace (3)
Struct Struct (23)
Enum Enum (10)
Interface Interface (11)
Function Function (12)
Method Method (6)
Constant Constant (14)
Struct field Field (8)
Enum variant EnumMember (22)
Type alias TypeParameter (26)
Local variable Not included in document symbols
Import Namespace (3)

12.3 Hierarchy

The tree reflects lexical nesting:

· Module → top-level declarations
· Struct → fields and methods
· Interface → methods
· Enum → variants
· Function → nested functions (if allowed)

Imports are shown as a flat list at the top level.

---

13. Workspace Symbols

13.1 workspace/symbol

Searches for symbols across the workspace.

Request:

```json
{ "query": "Poi" }
```

Response:

```json
[
  {
    "name": "Point",
    "kind": 23,
    "location": {
      "uri": "file:///home/user/project/src/geometry/point.vsp",
      "range": { "start": {"line": 5, "character": 0}, "end": {"line": 8, "character": 0} }
    },
    "containerName": "geometry.point"
  }
]
```

13.2 Query Semantics

The query is matched against:

1. Exact symbol names.
2. Prefixes of symbol names.
3. Fuzzy subsequences (e.g., Pnt matches Point).
4. Qualified names (e.g., geometry.point.Point).

Results are ranked by:

1. Exact match.
2. Prefix match at start of name.
3. Prefix match at start of any word in name.
4. Fuzzy match.
5. Workspace file proximity to the current file.

13.3 Rate Limiting

The server debounces workspace/symbol requests by 50ms. If multiple requests arrive within that window, only the latest is processed.

---

14. Rename and Prepare Rename

14.1 textDocument/prepareRename

Request:

```json
{
  "textDocument": { "uri": "file:///home/user/project/src/main.vsp" },
  "position": { "line": 10, "character": 8 }
}
```

Response:

```json
{
  "range": { "start": {"line": 10, "character": 6}, "end": {"line": 10, "character": 12} },
  "placeholder": "length"
}
```

If the symbol cannot be renamed (e.g., it is a keyword or an external symbol not writable by the user), the server returns null.

14.2 textDocument/rename

Request:

```json
{
  "textDocument": { "uri": "file:///home/user/project/src/main.vsp" },
  "position": { "line": 10, "character": 8 },
  "newName": "magnitude"
}
```

Response: A WorkspaceEdit:

```json
{
  "changes": {
    "file:///home/user/project/src/geometry/point.vsp": [
      {
        "range": { "start": {"line": 10, "character": 12}, "end": {"line": 10, "character": 18} },
        "newText": "magnitude"
      }
    ],
    "file:///home/user/project/src/main.vsp": [
      {
        "range": { "start": {"line": 10, "character": 6}, "end": {"line": 10, "character": 12} },
        "newText": "magnitude"
      }
    ]
  }
}
```

14.3 Rename Rules

· Renaming a local variable affects only its scope.
· Renaming a struct field affects all accesses and struct literals.
· Renaming a method affects all call sites and interface implementations.
· Renaming a type affects all declarations, type annotations, and constructors.
· Renaming a module requires updating all imports. The server may return additional edits for import statements.
· Renaming an external symbol (from another module not in the workspace) is rejected with an error.

14.4 Name Validation

The new name must be a valid Vesper identifier (see Vesper spec §4.3). If not, the server returns error -32803 (invalid rename).

14.5 Conflicts

If the new name conflicts with an existing symbol in the same scope, the server returns a warning in the edit's changeAnnotations (LSP 3.16+):

```json
{
  "changeAnnotations": {
    "conflict": {
      "label": "renaming may shadow existing symbol `magnitude`",
      "needsConfirmation": true
    }
  }
}
```

The client may prompt the user before applying.

---

15. Code Actions

15.1 textDocument/codeAction

Provides quick fixes, refactorings, and source actions.

Request:

```json
{
  "textDocument": { "uri": "file:///home/user/project/src/main.vsp" },
  "range": { "start": {"line": 10, "character": 0}, "end": {"line": 10, "character": 20} },
  "context": {
    "diagnostics": [ { "code": "VSP-W001", "message": "unused variable `x`" } ],
    "only": ["quickfix"]
  }
}
```

Response:

```json
[
  {
    "title": "Remove unused variable `x`",
    "kind": "quickfix",
    "diagnostics": [ { "code": "VSP-W001" } ],
    "edit": {
      "changes": {
        "file:///home/user/project/src/main.vsp": [
          {
            "range": { "start": {"line": 10, "character": 0}, "end": {"line": 10, "character": 20} },
            "newText": ""
          }
        ]
      }
    },
    "isPreferred": true
  }
]
```

15.2 Code Action Kinds

Kind Description
quickfix Fix a diagnostic
refactor.extract Extract expression into variable or function
refactor.inline Inline variable or function
refactor.rewrite Change code structure
source.organizeImports Sort and remove unused imports
source.fixAll Apply all quickfixes for the file

15.3 Quickfixes

Diagnostic Quickfix
VSP-E001 (undefined variable) "Did you mean xs?"
VSP-E003 (undefined function) "Did you mean print?"
VSP-E006 (interface not satisfied) "Add missing method foo"
VSP-E007 (immutable reassign) "Change let to var"
VSP-E008 (non-exhaustive match) "Add missing cases"
VSP-W001 (unused variable) "Remove" or "Rename to _"
VSP-W002 (unused import) "Remove import"
VSP-W003 (shadowing) "Rename shadowing variable"

15.4 Refactorings

Refactor Trigger
Extract to variable Select an expression
Extract to function Select statements
Inline variable Cursor on a variable declared with a simple initializer
Convert if/else to match Cursor on an if expression
Convert pipe chain to nested calls Cursor on a pipe chain
Wrap in try/catch Cursor on a statement

15.5 Source Actions

Action Effect
source.organizeImports Sort imports alphabetically, remove unused, group by standard/external/local
source.fixAll Apply all safe quickfixes in the file

15.6 codeAction/resolve

The server may return code actions with unresolved edits and provide them on codeAction/resolve. This is used for expensive operations such as source.fixAll on large files.

---

16. Code Lens

16.1 textDocument/codeLens

Returns code lens items for a document.

Response:

```json
[
  {
    "range": { "start": {"line": 12, "character": 0}, "end": {"line": 12, "character": 34} },
    "data": { "kind": "references", "symbol": "length" },
    "command": {
      "title": "3 references",
      "command": "editor.action.showReferences",
      "arguments": [ ... ]
    }
  }
]
```

16.2 Supported Lenses

Lens Description
References Number of references to a function, method, or type
Implementations Number of implementations of an interface method
Test coverage Not supported in v0.1

16.3 Resolution

The server may return a lightweight code lens with data and no command. The client sends codeLens/resolve to obtain the full item, including the count. This keeps codeLens responses fast for large files.

---

17. Formatting

17.1 textDocument/formatting

Request:

```json
{
  "textDocument": { "uri": "file:///home/user/project/src/main.vsp" },
  "options": {
    "tabSize": 4,
    "insertSpaces": true,
    "trimTrailingWhitespace": true,
    "insertFinalNewline": true,
    "trimFinalNewlines": true
  }
}
```

Response: TextEdit[] covering the whole document, or null if no changes.

17.2 Formatting Rules

vs-lsp follows the canonical Vesper style guide (vsfmt):

· Indentation: 4 spaces, no tabs.
· Line length: 100 characters (soft limit).
· One statement per line.
· Blank line between top-level declarations.
· No trailing whitespace.
· Exactly one newline at end of file.
· Space after , and : in expressions.
· No space inside parentheses or brackets.
· Space around binary operators.
· No space around . and ?.
· Opening brace always on the same line as its construct.

17.3 textDocument/rangeFormatting

Formats only the selected range. The server still parses the entire file to preserve syntactic context but produces edits only within the range.

17.4 textDocument/onTypeFormatting

Triggered on typing \n, :, and }:

· On \n: reindent the current line according to the block level.
· On :: indent the following line.
· On }: dedent the line to match the opening delimiter.

17.5 Idempotency

Formatting must be idempotent: applying formatting to an already-formatted document returns no edits.

---

18. Semantic Tokens

18.1 textDocument/semanticTokens/full

Returns semantic tokens for the entire document.

Response:

```json
{
  "resultId": "1",
  "data": [
    5, 0, 5, 23, 0,
    5, 7, 1, 8, 0,
    6, 4, 1, 8, 0,
    ...
  ]
}
```

The data array encodes tokens in groups of five integers:

1. Delta line from the previous token.
2. Start character on that line.
3. Length.
4. Token type (index into the legend).
5. Token modifiers (bitmask).

18.2 Token Types

See §3.2.1 for the full legend. The following mappings apply:

Vesper entity Token type
Module name namespace
Struct struct
Enum enum
Interface interface
Function function
Method method
Local variable variable
Struct field property
Enum variant enumMember
Type parameter typeParameter
Constant variable + readonly
Atom atom
Keyword keyword
Number number
String string
Comment comment
Operator operator
Function parameter parameter

18.3 Token Modifiers

Modifier Applies to
declaration The declaring occurrence of a symbol
definition Same as declaration for Vesper
readonly let bindings, const
mutable var bindings
static Module-level symbols
deprecated Symbols marked @deprecated
defaultLibrary Symbols from std.*
exported Symbols marked pub
documentation Symbols with doc comments
embedded Promoted fields from embedded structs

18.4 textDocument/semanticTokens/full/delta

If the client supports delta, the server returns only changes since the previous result.

Response:

```json
{
  "resultId": "2",
  "edits": [
    {
      "start": 10,
      "deleteCount": 5,
      "data": [ ... ]
    }
  ]
}
```

If the delta is larger than the full token list, the server returns null and the client falls back to full.

18.5 Incremental Updates

Semantic tokens are computed incrementally: only the changed lines are re-tokenized, and the delta is computed by comparing old and new token streams.

---

19. Inlay Hints

19.1 textDocument/inlayHint

Provides inline annotations: types of variables, parameter names at call sites.

Response:

```json
[
  {
    "position": { "line": 10, "character": 12 },
    "label": ": i32",
    "kind": 1,
    "paddingLeft": false,
    "paddingRight": true,
    "data": { "kind": "type", "symbol": "x" }
  },
  {
    "position": { "line": 15, "character": 8 },
    "label": "name:",
    "kind": 2,
    "paddingRight": true,
    "data": { "kind": "parameter", "argIndex": 0 }
  }
]
```

19.2 Hint Kinds

Kind LSP value Description
Type 1 Type of an inferred variable
Parameter 2 Parameter name at call site
Other 0 Miscellaneous

19.3 Type Hints

Shown for let/var bindings where the type is inferred:

```
let x = compute()      # shown as: let x: i32 = compute()
```

Hidden if the type annotation is explicit.

19.4 Parameter Hints

Shown at call sites for arguments whose meaning is unclear (e.g., multiple positional arguments of the same type):

```
create_user("Alice", 30, true)
        # shown as: create_user(name: "Alice", age: 30, active: true)
```

Hidden if the argument is a variable with the same name as the parameter.

19.5 inlayHint/resolve

Lazy resolution for expensive hints. The server may return a hint with only position and data, and fill in the label on inlayHint/resolve.

19.6 Configuration

Inlay hints may be disabled per-hint-kind via initializationOptions.vesper.inlayHints:

```json
{
  "types": true,
  "parameters": true,
  "rangeVariables": false,
  "chainedCalls": false
}
```

---

20. Folding Ranges

20.1 textDocument/foldingRange

Returns foldable regions.

Response:

```json
[
  { "startLine": 5, "endLine": 12, "kind": "region" },
  { "startLine": 20, "endLine": 25, "kind": "comment" },
  { "startLine": 30, "endLine": 35, "kind": "imports" }
]
```

20.2 Fold Kinds

Entity Kind
Function body region
Struct body region
Enum body region
Interface body region
Match expression region
Multi-line comment comment
Import block imports
Consecutive let/var declarations region

20.3 Fold Behavior

The server must respect the lineFoldingOnly client capability: if true, folding ranges are only lines, not character ranges.

---

21. Selection Ranges

21.1 textDocument/selectionRange

Given a list of positions, returns a chain of enclosing ranges.

Request:

```json
{
  "textDocument": { "uri": "file:///home/user/project/src/main.vsp" },
  "positions": [ { "line": 10, "character": 8 } ]
}
```

Response:

```json
[
  {
    "range": { "start": {"line": 10, "character": 6}, "end": {"line": 10, "character": 12} },
    "parent": {
      "range": { "start": {"line": 10, "character": 4}, "end": {"line": 10, "character": 18} },
      "parent": {
        "range": { "start": {"line": 10, "character": 0}, "end": {"line": 12, "character": 20} },
        "parent": {
          "range": { "start": {"line": 0, "character": 0}, "end": {"line": 50, "character": 0} }
        }
      }
    }
  }
]
```

21.2 Range Hierarchy

The chain grows outward:

1. Identifier.
2. Full expression.
3. Full statement.
4. Enclosing block (function body, if body, etc.).
5. Whole function or top-level declaration.
6. Whole document.

---

22. Document Highlight

22.1 textDocument/documentHighlight

Returns ranges of all occurrences of the symbol at the cursor, within the same file.

Response:

```json
[
  {
    "range": { "start": {"line": 10, "character": 6}, "end": {"line": 10, "character": 12} },
    "kind": 3
  },
  {
    "range": { "start": {"line": 15, "character": 8}, "end": {"line": 15, "character": 14} },
    "kind": 2
  }
]
```

22.2 Highlight Kinds

Kind Value Meaning
Text 1 Generic occurrence
Read 2 Read access
Write 3 Write access

---

23. Call Hierarchy

23.1 textDocument/prepareCallHierarchy

Prepares a call hierarchy for the symbol at the cursor. Returns the symbol as a call hierarchy item.

Response:

```json
[
  {
    "name": "length",
    "kind": 6,
    "uri": "file:///home/user/project/src/geometry/point.vsp",
    "range": { "start": {"line": 10, "character": 0}, "end": {"line": 11, "character": 20} },
    "selectionRange": { "start": {"line": 10, "character": 12}, "end": {"line": 10, "character": 18} },
    "data": { "symbolId": "point.length" }
  }
]
```

23.2 callHierarchy/incomingCalls

Returns all call sites that call this function.

23.3 callHierarchy/outgoingCalls

Returns all functions called by this function.

23.4 Recursion

Both incoming and outgoing calls may be recursive. The server detects cycles and does not recurse infinitely.

---

24. Type Hierarchy

24.1 textDocument/prepareTypeHierarchy

Prepares a type hierarchy for the type at the cursor.

24.2 typeHierarchy/supertypes

Returns the types this type extends via embedding. For interfaces, returns the interfaces this interface extends.

24.3 typeHierarchy/subtypes

Returns the types that embed this type, or the types that implement this interface.

---

25. Workspace Folders and Configuration

25.1 Workspace Folders

The server supports multiple workspace folders. Each folder is indexed independently.

workspace/didChangeWorkspaceFolders notifies the server of added and removed folders:

· On add: index the folder's files, publish diagnostics.
· On remove: discard index entries, clear diagnostics.

25.2 workspace/didChangeConfiguration

The client sends changed configuration.

```json
{
  "settings": {
    "vesper": {
      "formatting": {
        "indentWidth": 4,
        "maxLineLength": 100
      },
      "inlayHints": {
        "types": true,
        "parameters": true
      }
    }
  }
}
```

Server applies changes to affected documents and re-publishes diagnostics and semantic tokens if needed.

25.3 workspace/configuration

Server may request configuration from the client:

```json
{
  "items": [
    { "section": "vesper" },
    { "section": "vesper.formatting" }
  ]
}
```

25.4 workspace/didChangeWatchedFiles

Client notifies the server of file changes on disk:

· Created: index the new file.
· Changed: re-index; if the file is a dependency, re-diagnose dependents.
· Deleted: remove from index; report errors in files that imported it.

25.5 File Watching

The server registers file watchers via client/registerCapability:

```json
{
  "registrations": [
    {
      "id": "vesper-watch-vsp",
      "method": "workspace/didChangeWatchedFiles",
      "registerOptions": {
        "watchers": [
          { "globPattern": "**/*.vsp" },
          { "globPattern": "**/vsp.toml" }
        ]
      }
    }
  ]
}
```

---

26. Custom Extensions

vs-lsp defines the following Vesper-specific extensions. Clients that do not support them may ignore them.

26.1 vesper/compileFile

Compiles the current file to ARC assembly and reports any errors.

Request:

```json
{
  "textDocument": { "uri": "file:///home/user/project/src/main.vsp" }
}
```

Response:

```json
{
  "asmUri": "file:///home/user/project/build/main.asm",
  "diagnostics": []
}
```

26.2 vesper/showArcAssembly

Returns the generated ARC assembly for a given position's enclosing function.

Request:

```json
{
  "textDocument": { "uri": "file:///home/user/project/src/main.vsp" },
  "position": { "line": 10, "character": 8 }
}
```

Response:

```json
{
  "assembly": "sum:\n    enter #8\n    ...",
  "range": { "start": {"line": 10, "character": 0}, "end": {"line": 15, "character": 20} }
}
```

26.3 vesper/explainType

Provides a detailed explanation of a type, including its ARC representation.

Request:

```json
{
  "textDocument": { "uri": "file:///home/user/project/src/main.vsp" },
  "position": { "line": 10, "character": 8 }
}
```

Response:

```json
{
  "type": "Point",
  "representation": "struct with fields { x: f64 @ offset 0, y: f64 @ offset 8 }",
  "sizeBytes": 16,
  "methods": ["length", "translate"]
}
```

26.4 vesper/formatDocumentWithConfig

Like textDocument/formatting, but accepts per-request configuration overrides:

```json
{
  "textDocument": { "uri": "file:///home/user/project/src/main.vsp" },
  "options": {
    "indentWidth": 2,
    "maxLineLength": 80
  }
}
```

26.5 vesper/profileFunction

Not implemented in v0.1. Reserved for future use.

---

27. Performance and Caching

27.1 Latency Requirements

Operation Target latency
hover < 50ms
completion (initial) < 100ms
completion (resolved) < 50ms
definition < 50ms
references < 200ms (small workspace)
rename < 300ms (small workspace)
formatting < 100ms per 1000 lines
semanticTokens/full < 100ms per 1000 lines
codeAction < 100ms
documentSymbol < 50ms

27.2 Caching Layers

1. Lexer cache: per-file token stream, invalidated on change.
2. Parser cache: per-file AST, invalidated on change.
3. Type cache: per-file type information, invalidated on change to the file or its dependencies.
4. Symbol cache: workspace index, updated incrementally.
5. Semantic token cache: per-file token stream, with delta computed from AST change.

27.3 Memory Budget

The server targets 500 MB resident memory for workspaces up to 10 000 files. Beyond that, the server may reduce caching (e.g., keep only open files in memory) and report the reduction via window/logMessage.

27.4 Incremental Parsing

The parser uses a token-tree-based algorithm: it reuses subtrees that did not change. A file of 1000 lines requires less than 5ms for an incremental re-parse after a single-character edit.

27.5 Workspace Indexing

The workspace index is built lazily:

1. On initialize, if the workspace is small (< 500 files), index all files.
2. For larger workspaces, index only files that are open or imported by open files.
3. Index remaining files in the background, prioritized by proximity to open files.

Progress is reported via $/progress.

---

28. Error Handling

28.1 LSP Error Codes

Code Meaning
-32700 Parse error (invalid JSON)
-32600 Invalid request
-32601 Method not found
-32602 Invalid params
-32603 Internal error
-32800 Request cancelled
-32801 Content modified
-32802 Server cancelled
-32803 Request failed

vs-lsp uses -32803 for semantic failures (e.g., rename that cannot be applied).

28.2 Internal Errors

If the server encounters an internal error:

1. Log the stack trace to stderr.
2. Respond with error code -32603 and a human-readable message.
3. Do not crash the server. If the error is unrecoverable, terminate gracefully and let the client restart.

28.3 Recovered from Crashes

The server persists a lightweight state (open document URIs and versions) to disk. On restart, the client resends didOpen for all open documents, and the server rebuilds the index.

28.4 Compiler Failures

If the internal parser or type checker fails on a file (a bug), the server:

1. Reports the failure via window/showMessage with severity Error.
2. Publishes an empty diagnostic list for that file.
3. Continues operating on other files.

---

29. Security

29.1 Trust Boundary

The server reads files within the workspace folder. It must not read files outside the workspace unless explicitly requested by the user (e.g., a definition jump into the standard library).

29.2 Path Validation

All URIs received from the client are validated:

· Must be file:// URIs.
· Must be under one of the workspace folders.
· Path traversal (..) is not permitted.

29.3 Sandboxing

If the client provides a workspace trust capability, the server respects it:

· Untrusted workspace: the server operates in read-only mode, no code execution.
· Trusted workspace: the server may invoke the Vesper compiler for advanced features.

29.4 No Network Access

The server does not make network requests in v0.1. Future versions may fetch documentation or update packages, but only with explicit user consent.

29.5 No Code Execution

The server never executes user code. It compiles to AST and analyzes it statically. Diagnostics and semantic information are derived from this static analysis.

---

30. Version Compatibility

30.1 LSP Version

vs-lsp implements LSP 3.17. It is backward-compatible with LSP 3.16 clients, degrading gracefully:

· If the client lacks inlayHintProvider, no inlay hints are sent.
· If the client lacks semanticTokensProvider, no semantic tokens are sent.
· If the client lacks workspaceFolders, only the root URI is used.

30.2 Vesper Version

vs-lsp targets Vesper language version 0.1. It declares its supported version via serverInfo.version. Clients may warn users if the workspace uses a newer language version.

30.3 Feature Negotiation

The server inspects client.capabilities at initialization and enables only the features the client supports. Unsupported features are omitted from the server's capabilities response.

30.4 Deprecation

Deprecated features are marked in this specification and kept for at least one minor version cycle after deprecation. Removal is announced in the change log.

---

Appendix A: Message Sequence Examples

A.1 Initialization and First Diagnostics

```
Client → Server: initialize
Server → Client: initialize result (capabilities)
Client → Server: initialized
Client → Server: textDocument/didOpen (main.vsp)
Server → Client: textDocument/publishDiagnostics (main.vsp)
Client → Server: textDocument/didOpen (point.vsp)
Server → Client: textDocument/publishDiagnostics (point.vsp)
Server → Client: textDocument/publishDiagnostics (main.vsp)   ← updated due to new dependency
```

A.2 Hover and Definition

```
Client → Server: textDocument/hover
Server → Client: hover result
Client → Server: textDocument/definition
Server → Client: definition result
```

A.3 Rename

```
Client → Server: textDocument/prepareRename
Server → Client: prepare result (range, placeholder)
Client → Server: textDocument/rename
Server → Client: workspace edit
```

A.4 Code Action

```
Client → Server: textDocument/codeAction
Server → Client: code actions
Client → Server: codeAction/resolve (for the chosen action)
Server → Client: resolved action with full edit
```

---

Appendix B: Configuration Schema

The full configuration schema for vesper.* settings:

```json
{
  "$schema": "https://vesper.dev/schemas/vs-lsp-config.json",
  "type": "object",
  "properties": {
    "vesper.compilerPath": {
      "type": "string",
      "description": "Path to the Vesper compiler (vsc). Used for advanced analysis."
    },
    "vesper.arcAsPath": {
      "type": "string",
      "description": "Path to arc-as. Used for assembly preview."
    },
    "vesper.arcVmPath": {
      "type": "string",
      "description": "Path to arc-vm. Used for run/debug integration."
    },
    "vesper.formatting.indentWidth": {
      "type": "integer",
      "default": 4,
      "minimum": 1,
      "maximum": 8
    },
    "vesper.formatting.maxLineLength": {
      "type": "integer",
      "default": 100,
      "minimum": 60
    },
    "vesper.formatting.trimTrailingWhitespace": {
      "type": "boolean",
      "default": true
    },
    "vesper.formatting.insertFinalNewline": {
      "type": "boolean",
      "default": true
    },
    "vesper.inlayHints.types": {
      "type": "boolean",
      "default": true
    },
    "vesper.inlayHints.parameters": {
      "type": "boolean",
      "default": true
    },
    "vesper.inlayHints.chainedCalls": {
      "type": "boolean",
      "default": false
    },
    "vesper.diagnosticsOnChange": {
      "type": "string",
      "enum": ["immediate", "debounced", "save"],
      "default": "debounced"
    },
    "vesper.diagnosticsDebounceMs": {
      "type": "integer",
      "default": 200,
      "minimum": 0,
      "maximum": 2000
    },
    "vesper.maxDiagnosticsPerFile": {
      "type": "integer",
      "default": 200,
      "minimum": 1
    },
    "vesper.semanticTokens.enabled": {
      "type": "boolean",
      "default": true
    },
    "vesper.codeLens.references": {
      "type": "boolean",
      "default": true
    },
    "vesper.completion.autoImport": {
      "type": "boolean",
      "default": true,
      "description": "Automatically add imports for completion items."
    },
    "vesper.completion.fuzzyMatching": {
      "type": "boolean",
      "default": true
    },
    "vesper.workspaceIndexingMode": {
      "type": "string",
      "enum": ["eager", "lazy", "manual"],
      "default": "lazy"
    }
  }
}
```

---

Appendix C: Change Log

Version 0.1 (draft)

· Initial specification.
· LSP 3.17.
· All standard text synchronization, diagnostics, completion, hover, signature help, go-to-definition, references, document/workspace symbols, rename, code actions, code lens, formatting, semantic tokens, inlay hints, folding, selection ranges, document highlight, call hierarchy, type hierarchy.
· Vesper-specific extensions: vesper/compileFile, vesper/showArcAssembly, vesper/explainType, vesper/formatDocumentWithConfig.
· Configuration schema.

---

End of specification.
