# Python API

The `fourier` package exposes the compiler as a single function plus
the underlying lex / parse / codegen entry points.

## `compile_source`

```python
from fourier import compile_source

def compile_source(source: str, name: str | None = None) -> bytes
```

The full pipeline: lex → parse → codegen → assemble. Returns the raw
VM bytecode ready to deploy.

```python
src = '''
contract Counter {
    storage value: uint @ 0;
    fn init() { value = 42; }
    pub fn get() -> uint { return value; }
    pub fn inc() { value = value + 1; }
}
'''
bytecode = compile_source(src)
print(len(bytecode), "bytes")
# 200 bytes (approximate)
```

### Multi-contract sources

A source file may declare multiple top-level `contract` blocks. To
pick which one `compile_source` returns, pass `name=`:

```python
src = '''
contract Math   { pub fn add(a: uint, b: uint) -> uint { return a + b; } }
contract String { pub fn len(s: uint) -> uint { return 0; } }
'''
bc_math   = compile_source(src, name="Math")
bc_string = compile_source(src, name="String")
```

Without `name=`, a multi-contract source raises `ValueError`. For all
bytecodes at once, use `compile_source_all`:

```python
from fourier import compile_source_all

result = compile_source_all(src)
# {'Math': b'...', 'String': b'...'}
```

Raises:

- `LexError` on illegal tokens
- `ParseError` on malformed syntax
- `CompileError` on type / slot / unknown-identifier errors

See [Error reference](errors.md).

## Lower-level entry points

For tools that need to inspect intermediate stages:

```python
from fourier import tokenize, Token, TokenKind
from fourier import parse, ParseError
from fourier import compile_contract, CompileError
```

### `tokenize(source: str) -> list[Token]`

Lex only. Returns the full token list plus a final `Token(EOF, ...)`.
Each `Token` has `kind: TokenKind`, `value: object`, `line: int`,
`col: int`. See `fourier/lexer.py` for the full enum.

```python
tokens = tokenize("contract X { storage y: uint @ 0; }")
for t in tokens:
    print(t)
# Tok(CONTRACT @1:1)
# Tok(IDENT, 'X' @1:10)
# Tok(LBRACE @1:12)
# ...
```

### `parse(tokens) -> Contract | list[Contract]`

Recursive-descent parser. Returns a `Contract` AST node for a single-
contract source, or a `list[Contract]` for multi-contract sources —
see `fourier/ast_nodes.py` for the dataclass shapes. The companion
`parse_all(tokens) -> list[Contract]` always returns a list.

```python
ast = parse(tokens)
print(ast.name)              # 'X'
print(ast.storage)           # [StorageDecl(name='y', ty=TypeRef(name='uint'), slot=0, ...)]
print(ast.functions)         # []
```

### `compile_contract(contract: Contract) -> bytes`

Lower the AST to bytecode. Internally calls `vm.asm.assemble`.

```python
bytecode = compile_contract(ast)
```

## Combining stages

```python
from fourier import tokenize, parse, compile_contract

def trace_compile(source: str) -> bytes:
    tokens = tokenize(source)
    print(f"tokenized to {len(tokens)} tokens")
    ast = parse(tokens)
    print(f"parsed contract: {ast.name} ({len(ast.functions)} fns)")
    return compile_contract(ast)
```

This is what `compile_source` does, minus the prints.

## Error handling

```python
from fourier import compile_source
from fourier.lexer import LexError
from fourier.parser import ParseError
from fourier.codegen import CompileError

try:
    bc = compile_source(src)
except LexError as e:
    print(f"lex error at {e.line}:{e.col}: {e}")
except ParseError as e:
    print(f"parse error: {e}")
except CompileError as e:
    print(f"codegen error: {e}")
```

All three error classes carry the source position in their message
(format: `line:col: message`).

## Threading model

No global state. `compile_source` is pure (modulo `int.from_bytes` /
`hashlib` calls), safe to call concurrently. Each invocation creates
its own `_ContractGen` and `_FnCtx` objects.

## Caching

The compiler does no caching. If you compile the same source twice you
get the same bytecode twice; if you need a cache, build one on top of
`compile_source(src) → bytes`. The result is deterministic for a given
source string.

## Embedding in a build tool

Example: a tiny make-style helper that recompiles changed `.fou` files:

```python
import os, sys, time
from fourier import compile_source

SOURCES = ["counter.fou", "token.fou"]
OUTDIR = "build"

os.makedirs(OUTDIR, exist_ok=True)

for src_path in SOURCES:
    out_path = os.path.join(OUTDIR, os.path.basename(src_path) + ".bin")
    if (os.path.exists(out_path)
            and os.path.getmtime(out_path) >= os.path.getmtime(src_path)):
        continue
    print(f"compiling {src_path}...")
    with open(src_path) as f:
        bc = compile_source(f.read())
    with open(out_path, "wb") as f:
        f.write(bc)
    print(f"  wrote {len(bc)} bytes → {out_path}")
```
