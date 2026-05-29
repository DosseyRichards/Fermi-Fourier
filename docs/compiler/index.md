# Compiler

The Fourier compiler lives at `fourier/` in the source tree. It's a
single Python package with one public entry point:

```python
from fourier import compile_source

bytecode: bytes = compile_source(source_text)
```

That's the whole API surface. Everything else is internal.

## Pipeline

```text
.fou source text
      │
      │   tokenize()                  fourier/lexer.py
      ▼
list[Token]
      │
      │   parse()                     fourier/parser.py
      ▼
Contract AST
      │
      │   compile_contract()          fourier/codegen.py
      ▼
list of (op, arg) tuples + labels
      │
      │   assemble()                  vm/asm.py
      ▼
bytecode (bytes)
```

No optimizer, no IR, no register allocator. Each statement compiles to
a fixed bytecode template.

## Stage summary

| Stage | Input | Output | Errors raised |
|---|---|---|---|
| Lex | source string | `list[Token]` | `LexError` (unexpected char, empty hex literal) |
| Parse | tokens | `Contract` AST | `ParseError` (expected X got Y, etc.) |
| Codegen | AST | assembly items | `CompileError` (unknown identifier, slot collision, type mismatch, etc.) |
| Assemble | items | bytecode | `ValueError` (PUSH out of range, unknown asm item) |

See [Error reference](errors.md) for the catalog of error messages
each stage can produce.

## CLI

There is no standalone CLI in v1. You compile from Python:

```python
from fourier import compile_source

with open("counter.fou") as f:
    src = f.read()

bytecode = compile_source(src)

with open("counter.bin", "wb") as f:
    f.write(bytecode)
```

A standalone `fourierc` binary is on the
[follow-up list](https://github.com/DosseyRichards/Fermi-Fourier/blob/main/TODO.md).
Until it lands, use the `python -c '…'` one-liners on the
[CLI usage](cli.md) page — they're terminal-friendly and produce
identical bytecode.

## Subpages

- [**CLI usage**](cli.md) — how to compile a `.fou` file without
  writing Python
- [**Python API**](python.md) — `compile_source`, plus the internal
  hooks if you need lex/parse/codegen access
- [**Error reference**](errors.md) — every error message the compiler
  can produce, with cause and fix
