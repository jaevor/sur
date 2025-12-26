# Compiler pipeline

The Sur compiler must be resilient, incremental, and parallel.

## 1. Source

###### `surc_source`

All `.sur` files in the source-root folder are flattened into a `FileSet` is
created. An `IndexedFileSet` can then be made where each files' first few bytes
are read in order to organize them by module. They can then be queried to get
`Source<'s>`, which is a UTF-8 byte slice of the source code.

Notes:

- Caching can be done at this level by hashing file content.

## 2. Token

###### `surc_tokenizer`

The tokenizer parses each set of module files into a `TokenStream`.
Invalid/unexpected characters and newlines in strings are reported. The
`surc_tokenizer` crate does not allocate and is `no_std` (except in tests).

Notes:

- Invalid tokens are still put out, no errors are reported from the tokenizer.
- Certain code patterns can be "caught" directly by the tokenizer, such as `()`
  being tokenized into a `Token::Unit` _rather_ than the individual right `)`
  and left `(` parentheses.
- In the future when more Sur source code is written, analytics must be
  collected to better optimize and predict allocation size for tokens before
  tokenization begins.

## 3. Abstract Syntax Tree (`AST`)

###### `surc_ast`

The parser reads `TokenStream`s and parses them into abstract syntax trees. The
AST preserves everything about the original source code.

This representation is can be used for general tooling such as code formatting or syntax highlighting.

Notes:

- **The parser must try to be as optimistic as possible, only aborting further
  parsing if continuing isn't possible or does not make sense. As many errors
  as possible must be reported.**
- All invalid syntax patterns are reported from here.
- Literals are not stored directly, only their beginning and ending indexes in
  the source
- The root AST structure can contain a fingerprint field that is a hash of the
  contents of the source that it represents -- if hashes are the same, then the
  source does not need to be parsed into AST again (maybe this should be done at HIR level rather, because stuff like comments will change the hash but no anything that matters).

## 4. High-level Intermediate Representation (`HIR`)

###### `surc_hir`

This representation is essentially the same as the AST, but simplified in ways,
e.g,`(42)` would simplify to just `42`, and comments are removed. This
representation introduces:

- Scopes
- String interning
- The `typing_status` field on nodes

Notes:

- _HIR is only generated for the current project that is being compiled, not for
  any dependencies._
- After this stage, the `Source` of the code is no longer needed. \* (potentially AST too)

### 4.1 Type Solving

###### `surc_hir/solving`

Solving involves type inference, constraint resolution, and type checking. It
is a pass that goes over the HIR. Most errors about written code is reported
from here.

Notes:

- If a type mistmatch occurs, e.g. `let x: u8 = "foo"`, the assignment of a
  string to an integer is reported, but code that makes use of `x` later on will
  treat it as a valid `u8`. "Trivial" errors such as that must not stop the world.

## 5. Sur Intermediate Representation (**SIR**)

###### `surc_sir`

SIR is a graph representation of Sur code and is generated for each function and
constant body.

Things that are analyzed at this level:

- Ownership and borrowing
- Valid control flow
- **Checking `match` expressions**

## 6. Cranelift Intermediate Format (**CLIF**)

###### `surc_cl`

SIR is lowered into CLIF where the instructions undergo further optimizations.

## 6.1 Target machine code generation

After the CLIF is finalized, generation of machine code begins and a binary is
created.

<br><br><br>

# Notes

This project uses the `Vari` suffix instead of `Kind` for enums that represent
variants/kinds of something. For example, `LiteralVari` instead of
`LiteralKind`.

<br><br><br>

# Turing Incompleteness

Every loop must prove progress towards termination.
