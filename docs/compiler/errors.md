# Error reference

Every error message the compiler can produce, with the stage that
raises it and the typical cause.

All errors carry a source position in the form `line:col: message`.

## Lex errors (`LexError`)

Source: `fourier/lexer.py::tokenize`.

| Message | Cause |
|---|---|
| `empty hex literal` | `0x` with no hex digits following |
| `unexpected character: '<c>'` | A character that doesn't start any token (control chars, non-ASCII letters outside identifier rules, etc.) |

Fix: check the offending character; remove or replace with whitespace
/ a valid token.

## Parse errors (`ParseError`)

Source: `fourier/parser.py`.

Generic shape: `expected <KIND>` or `expected <category>`, with the
unexpected token kind shown.

| Pattern | Cause |
|---|---|
| `expected CONTRACT` | File doesn't start with `contract`. There must be exactly one top-level `contract Name { ... }` |
| `expected IDENT` after `contract` | Missing contract name |
| `expected LBRACE` | Missing `{` |
| `expected SEMI` | Missing `;` after a declaration or statement |
| `expected RBRACE` | Unbalanced braces |
| `expected COLON` | Missing `:` between name and type, e.g. `let x = 1` instead of `let x: uint = 1` |
| `expected AT` | Missing `@ N` slot annotation on a `storage` decl |
| `expected RPAREN` / `RBRACKET` | Unbalanced grouping |
| `expected storage/event/struct/fn` | Token at top level that isn't one of those keywords |
| `expected type` | `:` followed by something that isn't a type name |
| `expected expression` | A position needed an expression — got punctuation or EOF |
| `tuple binding has N names but M types` | `let (x, y): (uint, address, bool) = ...` mismatch |

Fix: the position pinpoints the offending token. Inspect the source
character-by-character against the [Quick reference](../quick-reference.md).

## Compile errors (`CompileError`)

Source: `fourier/codegen.py`. These check semantics — name resolution,
type compatibility, storage layout.

### Storage / slot errors

| Message | Cause |
|---|---|
| `storage slot N collides with reserved init-flag slot (2**256 - 1); choose a different slot` | A `storage` decl pinned slot `2^256 - 1`, which is reserved for the init guard |
| `storage slot N out of range [0, 2**256)` | Negative or huge slot literal |
| `storage slot N already used by 'NAME'` | Two `storage` decls share the same slot |
| `unknown struct type 'NAME'` | A storage decl references a struct that wasn't declared |

### `init` errors

| Message | Cause |
|---|---|
| `init function must not be declared 'pub' — it runs automatically on first call, never via external selector` | Wrote `pub fn init`; drop the `pub` |
| `init function must not take parameters in Fourier v1; use storage initialization or constants instead` | Wrote `fn init(a: uint)`; init takes no args |
| `init function must not return a value` | Wrote `fn init() -> uint`; init has no return |
| `return is not allowed inside init — init runs to its end and falls through to the dispatcher` | A `return` statement inside the init body; remove it |

### Name resolution

| Message | Cause |
|---|---|
| `unknown identifier 'NAME'` | A `Var` or assignment refers to a name that isn't a local or storage decl |
| `unknown function 'NAME/N'` | A `Call` whose name isn't a known builtin (no user-defined call mechanism in v1) |
| `unknown event 'NAME'` | `emit NAME(...)` with no matching `event NAME(...)` decl |
| `event 'NAME' expects K args, got N` | `emit` arg count mismatch |

### Type errors

| Message | Cause |
|---|---|
| `type mismatch: cannot apply '<op>' between 'address' and 'uint' (address values aren't arithmetic)` | Arithmetic or ordering op between address and non-address |
| `cannot assign to mapping 'NAME' without index` | `m = value` instead of `m[key] = value` |
| `mapping 'NAME' used as scalar` | Reading a mapping name without `[...]` |
| `'NAME' is not a storage variable` | Indexing or field-accessing a non-storage name |
| `'NAME' is not a storage mapping or array` | Index assignment to a scalar storage |
| `'NAME' is not a mapping or array` | Indexed read on a scalar storage |
| `'NAME' is not a struct type` | Field access on non-struct storage |
| `struct 'S' has no field 'F'` | Typo or missing field |
| `array 'NAME' takes one index, got N` | Multi-index access on an `array[T]` (use a single index) |
| `return with value in void function` | `return EXPR;` in a function with no `-> TYPE` |

### Tuple errors

| Message | Cause |
|---|---|
| `tuple literal only allowed in 'return (...)' or 'let (...) = (...)'` | A `(a, b)` expression anywhere except a return / destructuring let |
| `tuple destructuring requires a literal tuple RHS in v1 (let (x, y) = (a, b);)` | RHS isn't a `(...)` tuple literal — e.g. binding a call return |
| `tuple binding has N names but RHS has M elements` | Names / RHS arity mismatch |

### Builtin-specific errors

| Message | Cause |
|---|---|
| `push(...) requires a storage array as its first arg` (also `len`, `pop`) | First arg of `len` / `push` / `pop` isn't a bare storage-array name |
| `verify_sig(scheme_id, ...) requires scheme_id to be a literal int` | `scheme_id` passed as a variable / expression |
| `unknown crypto scheme id N; known: [1, 2]` | `verify_sig(N, ...)` for an N not in `CRYPTO_SCHEMES` |

### Field assignment errors

| Message | Cause |
|---|---|
| `field assignment on 'NAME': not a storage variable` | `obj.field = value` where `obj` isn't storage |

## Assembler errors (`ValueError`)

Source: `vm/asm.py`.

| Message | Cause |
|---|---|
| `PUSH value out of range: V` | Negative literal or value > `2**256` |
| `DUP i out of range: i` | `DUP` index outside 1..16 |
| `SWAP i out of range: i` | `SWAP` index outside 1..16 |
| `LOG n out of range: n` | More than 4 topics (LOG can encode 0..4) |
| `unknown asm item: <item>` | Internal codegen bug (please file an issue) |

These typically indicate a codegen bug rather than a user error — the
language doesn't expose ways to overflow these limits directly. If you
hit one, capture the source and report it.

## Common fixes

| Symptom | Likely fix |
|---|---|
| Parser dies at `LBRACE` after `fn name()` | Missing `->` and return type — but you wrote the body and the parser tried to read it as the type. Add `-> TYPE`. |
| `unknown identifier` for a name you definitely declared | Storage decls and `let`s aren't visible across functions, only locals within the function are. Check spelling. |
| `storage slot collision` after copying stdlib | Renumber the stdlib's `@ slot` declarations to avoid yours. See [Stdlib slot conventions](../stdlib/index.md#slot-conventions). |
| `mapping used as scalar` when emitting an event | Forgot the index: `emit X(balances[addr])`, not `emit X(balances)`. |
