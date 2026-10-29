# Design: Sans-I/O Parsing and Lossless Editing for Beancount

This document proposes a design for parsing, editing, and saving Beancount files, particularly in environments without direct filesystem access (e.g., a native library on mobile using tools like the Crux crates by Red Badger, or compiled to WebAssembly). It takes inspiration from the Crux architecture and the "Sans I/O" concepts popularized by Faster Than Lime.

## 1. Motivation

Currently, `beancount-parser-lima` is tightly coupled to `std::fs` through `BeancountSources`, which synchronously reads files and resolves globs during construction (see `try_read_with_includes` in `src/sources.rs`).
This poses two main problems:
1. **No Filesystem Access:** In contexts like WebAssembly or mobile libraries built via Crux, the core logic cannot (or should not) block on filesystem calls.
2. **Lossy Parsing:** The parser discards comments, whitespace, and non-semantic formatting (via `logos` regex definitions like `ignored_whole_line` and `comment_to_eol`). When edits are made to the AST and serialized back, the original formatting is lost, resulting in destructive changes to a user's Beancount file.

## 2. Sans I/O Parsing Architecture

### Concept
The "Sans I/O" philosophy advocates for separating protocol or parsing logic from I/O execution. In our context, parsing should be a pure state machine. The parser analyzes text, and if it encounters an `include` directive, it pauses and yields a side-effect "Request" to the outer shell.

This heavily aligns with the **Crux** architectural pattern (used by Red Badger), where the Core is pure and requests capabilities (like HTTP, or in our case, File I/O) from the Shell.

### Proposed Design
Introduce a new state machine for building sources, e.g., `BeancountSourcesBuilder`.

```rust
pub enum SourceRequest {
    /// Request to read a specific file path or expand a glob
    Include(String),
}

pub enum SourceState {
    /// The builder needs more files to continue parsing
    Awaiting(SourceRequest),
    /// The builder has completely resolved all includes
    Complete(BeancountSources),
    /// An error occurred that halts parsing
    Error(String),
}

pub struct BeancountSourcesBuilder { ... }

impl BeancountSourcesBuilder {
    pub fn new(root_content: String) -> Self { ... }

    /// Advance the state machine
    pub fn advance(self) -> SourceState { ... }

    /// Provide the response for an awaited include
    pub fn provide_include(self, content: Result<String, String>) -> Self { ... }
}
```

**Benefits:**
- **Wasm/Mobile Compatibility:** The Shell (Swift/Kotlin/JS) receives the `Awaiting(Include(path))` event, reads the file or fetches the string via whatever means necessary, and passes it back into `provide_include`.
- **Pure Core:** The parsing logic inside `BeancountSources` becomes purely synchronous and memory-bound.

## 3. Lossless Editing & Saving (Concrete Syntax Tree)

To support "parsing, editing, and saving those edits of just a single file" without destroying the user's formatting, the parser must transition from an Abstract Syntax Tree (AST) to a Concrete Syntax Tree (CST).

### The Problem in the Current Codebase
`beancount-parser-lima` uses `logos` and `chumsky`. In `src/lexer.rs`, formatting features like `comment_to_eol` and `ignored_whole_line` are parsed as whitespace or stripped out entirely. The output `Directive` contains only semantic data. Modifying a `Directive` and converting it back to a string would result in stripped comments and normalized whitespace.

### Proposed Solutions

#### Option A: CST via `rowan` (Recommended for pure editors)
To achieve true lossless editing, the parser would need to retain every single token, including whitespace, comments, and invalid syntax. This is typically done using libraries like `rowan` (used by `rust-analyzer` and `rnix`).

- **Pros:** Full fidelity. You can replace a single semantic node (e.g., an Account name), and the rest of the file (including inline comments) remains mathematically untouched.
- **Cons:** A massive rewrite of the parsing logic. Chumsky supports recovery, but mapping its output to an untyped green tree (like Rowan's) is non-trivial.

#### Option B: Span-based Delta Updates
The parser currently provides `Span`s for its `Directive`s. If we only need to edit *whole* directives (e.g., appending a posting or deleting a transaction):
1. Keep the original source string.
2. When a `Directive` is edited, format the *new* directive as a string.
3. Use the original `Directive`'s `Span` to splice the new string into the original source string.

- **Pros:** Can be implemented with the current `chumsky` architecture relatively easily.
- **Cons:** Generating the new directive string might look visually different from the user's formatting. For example, if a user aligns their numbers at column 50, the generated string might only use a single space.

## 4. Potential Challenges implementing this in the current codebase

If implementing this design in `beancount-parser-lima`, you would encounter several challenges:

1. **Glob Resolution without `std::fs`:**
   Currently, the parser uses `glob::glob_with` to expand included files. If filesystem access is removed (Sans-I/O), the Core can only ask the Shell to "resolve this glob". The Shell would need to implement its own globbing mechanism, which can be difficult in Javascript or Swift without pulling in heavy dependencies.

2. **File Paths vs URIs:**
   `BeancountSources` heavily relies on `std::path::PathBuf`. In a Wasm environment or a virtual filesystem, a URI or simple `String` identifier is more appropriate. Refactoring `PathBuf` out of `SourceId` mapping will touch a large portion of `src/sources.rs` and `src/lib.rs`.

3. **Duplicated Include Detection:**
   The parser currently canonicalizes paths (`canonicalize()`) to detect duplicate imports. Without a real filesystem, canonicalization is impossible. The Shell must be responsible for canonicalizing paths before returning them to the Core, or the Core must assume all paths provided by the Shell are already canonical.

## References
1. *Crux (Red Badger)*: Architectural pattern for cross-platform apps separating capabilities from pure core logic. (https://redbadger.github.io/crux/)
2. *Sans I/O*: "Writing network protocols in Python is hard" / "Sans-I/O" concepts by Faster Than Lime and the Python community. (https://sans-io.readthedocs.io/)
